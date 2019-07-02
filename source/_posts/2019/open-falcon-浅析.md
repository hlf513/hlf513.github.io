---
layout: "post"
title: "Open-falcon 浅析"
date: "2019-06-11 23:44"
tags:
  - falcon
categories:
  - 监控
---
# 监控系统概述

在监控系统领域，相信大家都经历过 Zabbix 的时代；在公司刚起步，机器数量不多时，Zabbix 可以很好的满足我们的需求，但是随着业务的发展，Zabbix 的存储会成为主要的性能瓶颈，从而引发很多问题，增加运维成本。

近些年来，随着互联网技术的不断发展，技术架构的不断演进，监控领域有两个我值得推荐的开源系统：一个是小米出品的 Open-Falcon，一个是基于 Google Borgmon 的开源实现 Prometheus。

今天主要介绍下 Open-Falcon。
<!-- more -->

Open-Falcon 有如下特点：
1. **强大灵活的数据采集**：自动发现，支持falcon-agent、snmp、支持用户主动push、用户自定义插件支持、opentsdb data model like（timestamp、endpoint、metric、key-value tags）
1. **水平扩展能力**：支持每个周期上亿次的数据采集、告警判定、历史数据存储和查询
1. **高效率的告警策略管理**：高效的portal、支持策略模板、模板继承和覆盖、多种告警方式、支持callback调用
1. **人性化的告警设置**：最大告警次数、告警级别、告警恢复通知、告警暂停、不同时段不同阈值、支持维护周期
1. **高效率的graph组件**：单机支撑200万metric的上报、归档、存储（周期为1分钟）
1. **高效的历史数据query组件**：采用rrdtool的数据归档策略，秒级返回上百个metric一年的历史数据
1. **dashboard**：多维度的数据展示，用户自定义Screen
1. **高可用**：整个系统无核心单点，易运维，易部署，可水平扩展
1. **开发语言**： 整个系统的后端，全部golang编写，portal和dashboard使用python编写。

简而言之：Open-falcon 是一个模块化、高可用、高性能、支持水平扩展的监控告警系统，支持机器监控、业务监控、各种开源软件的监控。

# 架构
![Architecture](/images/2019/06/architecture.png)

## Agent
> 数据采集组件

部署在业务机器上，主要作用：
1. 自动采集预先定义的各种采集项（机器级别的监控指标）
2. agent 还提供一个 HTTP 接口（/v1/push），用于接收用户自定义上报数据

每隔60秒，通过 JsonRPC push 数据到 Transfer 模块（使用长连接）。


## Transfer
> 数据转发服务

主要作用：
1. 接收 agent 上报的数据
2. 按照哈希规则进行数据分片，并将分片后的数据分别 push 给 graph、judge 等组件

## Graph
> 存储绘图数据、历史数据

主要作用：
1. 接口 transfer 推送数据
2. 处理 API 组件的查询请求、返回绘图数据

## API
> 绘图数据的查询接口

主要作用：根据一致性哈希算法去相应的 graph 实例查询不同监控项的数据，汇总后返回

## Dashboard
> 面向用户的查询界面

## Judge
> 告警判断

因为数据量太大，此组件放在 transfer 组件之后，这样每个 judge 只需要处理一小部分数据；主要作用：
1. 接口 transfer 推送数据
2. 分析数据，判断是否触发告警，需要告警则写入 redis

部署一个 judge 实例处理50万~100万数据，用个5G~10G内存。

## Alarm
> 处理告警事件

主要作用：
1. 从 redis 读取数据，触发动作（短信、邮件、回调等）
2. 告警合并
3. 已经发送的告警信息存入 MySQL，默认存7天

alarm是个单点，因为未恢复的告警是放到alarm的内存中的，alarm还需要做报警合并。需要做好存活监控。


## HBS
> 心跳服务器(Heartbeat Server)

至少部署两个实例以保证可用性，一般一个实例可以搞定5000台机器；主要作用：
1. 所有 agent 都会连到 HBS，每分钟发一次心跳请求，并告知 agent 应该采集哪些端口和进程
2. 维护业务机器的信息（host 表）
2. 告知 judge 报警策略

## Nodata
> 检测监控数据的上报异常

主要作用：配置了nodata的采集项超时未上报数据，nodata生成一条默认的模拟数据

## Aggregator
> 集群聚合

主要作用：聚合某集群下的所有机器的某个指标的值，提供一种集群视角的监控体验

## Task
> 定时任务

主要作用：
1. index更新。包括图表索引的全量更新 和 垃圾索引清理。
1. falcon服务组件的自身状态数据采集。定时任务采集了transfer、graph、task这三个服务的内部状态数据。
1. falcon自检控任务

# 设计理念
## 数据采集
1. 制定接口规范，以此接入各种监控数据
2. agent 自发现采集各种 Linux 性能指标，无需配置
3. 由 HBS 下发各种采集指标、策略
4. 支持 plugin；用户把插件提交到指定的 git repo，server端提供一个配置，哪些机器应该执行哪些插件，通过 HBS 把这个信息分发给 agent，agent 每隔一段时间去 git pull 这个 git repo，采集脚本就完成了分发。执行周期通过解析文件名来执行：60_action.sh，60s 执行一次。脚本执行完了，把输出打印到stdout，agent 截获之后 push 给 server

## Tag
Tag 是一种聚合手段，可以用更少的配置覆盖更多的监控项。例如：
``` js
{
    "endpoint": "qd-sadev-falcon-judge01.hd",
    "metric": "latency",
    "tags": "department=sadev,project=falcon,module=judge,method=falcon.judge.rpc.send",
    "value": 10.2,
    "timestamp": 1427204756,
    "step": 60,
    "counterType": "GAUGE"
}
```
如果我们这么配置：`latency/department=sadev all(#2) > 20`，意味着对sadev这个部门的所有接口的latency都做了策略配置。

## 模板继承

同一个部门的机器，根据不同的业务对监控策略的要求是不一样的，比如业务 A 复杂高，load.1min > 10 就报警，业务 B 复杂低，load.1min > 5 就报警。若不支持模板继承，则需要配置两份策略，而模板继承就减少了此类工作量。

## 数据存储
Open-falcon 把数据按照用途分成两类，一类是用来绘图的，一类是用户做数据挖掘的。关于绘图数据，在数据每次存入的时候，会自动进行采样、归档。我们的归档策略如下，历史数据保存5年。同时为了不丢失信息量，数据归档的时候，会按照平均值采样、最大值采样、最小值采样存三份。

对于原始数据，transfer会打一份到hbase，也可以直接使用opentsdb

# 使用示例

## 部署
参见[小米公司部署Open-Falcon的一些实践经验](http://book.open-falcon.org/zh_0_2/practice/deploy.html)

## 监控网络
![dashboard](/images/2019/06/dashboard.png)
![Screen](/images/2019/06/screen.png)

## 配置告警
![template](/images/2019/06/template.png)

## 上报接口状态
上报状态码、耗时等，可监控接口的健康、性能等。
![falcon](/images/2019/06/falcon.png)

# 参考资料
1. [Open-Falcon 官方文档](https://github.com/open-falcon/falcon-plus)

---
layout: "post"
title: "监控 Go runtime"
date: "2019-07-28 04:02"
tags:
  - runtime
  - graphite
  - grafana
categories:
  - Go
  - 监控
---

GO 通常作为守护进程存在，若我们不做任何监控，放任其「自由」的使用资源，那有可能会发现意外情况，比如「Goroutine 泄露」。那我们应该如何监控，又要监控哪些指标呢？

<!-- more -->

# 监控指标
监控的核心是监控指标，「有用」的指标才是有意义，那 Go 有哪些有意义的指标呢？

我认为只要监控`go runtime` 相关的指标就行，整体分为两大类：
1. CPU：goroutine、cgo
2. 内存：gc、heap、stack

# 监控架构
既然确定了监控指标，那如何监控呢？这里要感谢 Brian Hatfield 开源的 [go-runtime-metrics](https://github.com/bmhatfield/go-runtime-metrics)，我们可以利用它快速的搭建监控系统。

整体架构如下：
![Go runtime 监控架构图](/images/2019/07/架构图.png)

由图可知，我们使用了`Grafana`，`Graphite`，`Statsd`；其中`Statsd`是和运行程序部署在一起的，默认是10s 上报一次，对性能的影响不是太大，真的出现瓶颈的时候，可以再抽离；`Graphite` 是包含图表的；这里使用的是 `Grafana` 集成 `Graphite`。

## Go-runtime-metric
> https://github.com/bmhatfield/go-runtime-metrics

工作原理：
1. 程序启动后，开启一个 `goroutine` 采集上报 `metric`，数据收集使用 `statsd`，传输协议采用 `UDP`
2. 在 `main`函数中使用 `flag.Parse()` 才会打开采集上报（上报服务地址等通过命令行参数修改）
3. 采样率 100%，且不可修改（可以自己 fork 后修改）
4. 上报间隔 10s，可通过命令行`--pause`修改
5. 使用的 metric type 是 `Gauges`
6. 所有的采集项都是通过 `runtime` 包完成的；统计 `mem` 时，会触发 `STW`（若关心性能，建议关闭统计 `mem`）

可用参数：
```sh
--statsd=localhost:8125 # Statsd host:port
--metric-prefix # Metric prefix path；用于区分机器/应用等
--pause=10 # 上报间隔；default 10s
--publish-runtime-stats=true # 是否采集 go runtime；采集总开关
--cpu # 是否采集 CPU
--mem # 是否采集 mem
--gc # 是否采集 GC (必须开启 --mem)
```

## Statsd
> https://github.com/statsd/statsd

基于 node.js 实现，用于收集 metric 数据，支持 UDP、TCP 协议，默认每 10s 向后端发送一次数据，支持集群部署。支持的 [metric type](https://github.com/statsd/statsd/blob/master/docs/metric_types.md) 有 Counting、Sampling、Timing、Gauges。支持多种 [backend](https://github.com/statsd/statsd/blob/master/docs/backend.md)，内置三个：`Graphite`,`Console`,`Repeater`，第三方见文档。

启动前使用`run_tests.js` 检查下环境等问题。使用起来很简单，主要是两个配置文件：`exampleConfig.js`、`exampleProxyConfig.js`，前者用于单机部署，后者用于集群部署；详细配置含义见文件内容注释。

使用`Graphite`示例：
```js
// 单机部署
{
  graphitePort: 2003
, graphiteHost: "localhost"
, port: 8125
, server: "./servers/udp"
, debug: true
, backends: [ "./backends/console", "./backends/graphite" ]
}
```

## Graphite
> http://graphiteapp.org/

Graphite 主要作用是保存 metric 数据，并将其可视化。内置三个组件：
1. carbon：监听时间序列数据（守护进程）
2. whisper：存储时间序列数据（数据存储-类似 RRD）
3. graphite-webapp：渲染时间序列数据，提供 Web & API（Web 应用）

架构图如下：
![graphite 架构图](/images/2019/07/graphite-架构图.png)

### Carbon

Carbon 主要是监听并处理 metric 。

#### 配置
> https://graphite.readthedocs.io/en/latest/config-carbon.html

包含多个 conf（默认在 /opt/graphite/conf）：

1. **carbon.conf**
  主要是三个 sectoion：[cache] 缓存、[relay] 转发、[aggregator] 聚合

1. **storage-schemas.conf**
  主要是存储周期设置。修改后不会立即生效，需要执行下`whisper-resize.py`
  ``` ini
  [apache_busyWorkers]
  pattern = ^servers\.www.*\.workers\.busyWorkers$
  # 15s 一个点，存 7 天；1m 一个点，存 21 天...
  retentions = 15s:7d,1m:21d,15m:5y
  ```
1. **storage-aggregation.conf**
  主要是多个相同 metric 聚合 metric，修改后不会立即生效，需要执行`whisper-set-aggregation-method.py *.wsp <method>`
  ```ini
  [all_min]
  pattern = \.min$
  # 上一个 retention level 至少含有下一个 level 的 10% 的值
  xFilesFactor = 0.1
  # 聚合方法
  aggregationMethod = min
  ```
1. **relay-rules.conf**
  主要是转发数据
  ``` ini
  [example]
  pattern = ^mydata\.foo\..+
  servers = 10.1.2.3, 10.1.2.4:2004, myserver.mydomain.com
  ```

1. **aggregation-rules.conf**
  主要是根据 rule、method，对 metric 进行聚合操作
  ``` sh
  output_template (frequency) = method input_pattern
  ```

1. **rewrite-rules.conf**
  根据 rule 对 metric 进行重命名
  ```sh
  regex-pattern = replacement-text
  ```

1. **whitelist.conf**
  设置 metric 白名单

1. **blacklist.conf**
  设置 metric 黑名单

#### 启动

**单机**
> https://graphite.readthedocs.io/en/latest/admin-carbon.html

使用 `carbon-cache.py`，相关配置：carbon.conf、storage-schemas.conf
```sh
/opt/graphite/bin/carbon-cache.py start
```
若有性能问题，可以在运行`carbon-cache.py`之前使用`carbon-aggregator.py`，对数据进行聚合，减少数据量。相关配置：carbon.conf、aggregation-rules.conf


**集群**
> https://graphite.readthedocs.io/en/latest/carbon-daemons.html

使用 `carbon-relay.py`，主要作用是复制和分片。相关配置：carbon.conf、relay-rules.conf
也可以使用`carbon-aggregator-cache.py`,它合并了`carbon-aggregator.py`和`carbon-cache.py`两个脚本，相关配置：carbon.conf、relay-rules.conf、aggregation-rules.conf

### Graphite-web

使用 Django 开发。

#### 配置
> https://graphite.readthedocs.io/en/latest/config-local-settings.html

这里的 database 主要是存储用户、dashboard 等信息。默认使用 SQLite。

#### 启动
> https://graphite.readthedocs.io/en/latest/config-webapp.html

配置 nginx + gunicorn

### Whisper
> https://graphite.readthedocs.io/en/latest/whisper.html

存储时间序列数据，类似RRD，一个环形数据库。

# 部署

本地学习环境安装，非高性能版安装。

## 安装步骤
1. **安装 Graphite + statsd**
  使用 Docker 安装，推荐使用 Graphite 官方的 [docker repo](https://github.com/graphite-project/docker-graphite-statsd)，包含 Graphite & statsd

2. **安装 Grafana**
3. **Go 工程代码**中添加 `github.com/bmhatfield/go-runtime-metrics`

使用示例：
```go
package main

import (
	"flag"
	"log"
	"net/http"
	"os"

	_ "github.com/bmhatfield/go-runtime-metrics"
)

func main() {
  // 打开采集
	flag.Parse()

	cwd, err := os.Getwd()
	if err != nil {
		log.Fatal(err)
	}

	srv := &http.Server{
		Addr:    ":8000", // Normally ":443"
		Handler: http.FileServer(http.Dir(cwd)),
	}
	log.Fatal(srv.ListenAndServe())
}
```

# 参考资料
1. [Graphite](http://graphiteapp.org/)
2. [Statsd](https://github.com/statsd/statsd)
3. [Go-runtime-metric](https://github.com/bmhatfield/go-runtime-metrics)

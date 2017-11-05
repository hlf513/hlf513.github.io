---
layout: post
title: PHP 通过 stomp 协议和 ActiveMQ 通信
date: '2015-11-26 00:32:26 +0800'
tags:
  - php
  - activeMQ
---

# 什么是 activeMQ？
一个开源消息队列,使用 java 开发,属于 apache 基金会.  
官方网站:http://activemq.apache.org  
官方文档:http://activemq.apache.org/getting-started.html

# php 如何和 activeMQ 通信？
采用 stomp 协议  
php 安装 stomp 的扩展
``` php
pecl install stomp
```
采用第三方的类库:https://github.com/dejanb/stomp-php
# php 如何监控 activeMQ 中队列的状态？

activeMQ 需要打开StatisticsPlugin  
方法:http://activemq.apache.org/statisticsplugin.html

``` php
/**
 * 获取指定队列的待处理消息数量
 *
 * @param string $queue 待查询的队列名
 *
 * @return mixed 成功返回待处理的消息数量
 */
function getMQStatus($queue)
{
    $result = $num = FALSE;
    $statusqueue = "/queue/ActiveMQ.Statistics.Destination.{$queue}";//固定格式
    //$statusqueue = "/queue/ActiveMQ/Statistics/Destination/{$queue}";//固定格式

    //开启ActiveMQ
    $link = stomp_connect('tcp://192.168.221.129:6161');//broker 的地址
    if (!$link) {
        die("Can't connect MQ !!");
    }

    if (FALSE !== $link) {
        //查询之后的结果存放处
        $resultqueue = "/queue/test_status_{$queue}";

        //设定采用JSON格式
        stomp_subscribe($link, $resultqueue, array("transformation" => "jms-map-json"));
        //送出空字符串
        $result = stomp_send($link, $statusqueue, '', array("reply-to" => $resultqueue));
        if (FALSE === $result) {
            echo " send error" . PHP_EOL;
        }
        //取得状态
        while (stomp_has_frame($link)) {
            $frame = stomp_read_frame($link);

            if (FALSE != $frame) {
                stomp_ack($link, $frame['headers']['message-id']);
                $obj = json_decode($frame['body'], TRUE);//$obj 包含队列的所有信息
                //print_r($obj);
                //取得目前数量（尚可取得其他状态）
                foreach ($obj['map']['entry'] as $pitem) {
                    if ('size' == $pitem['string']) {
                        $num = $pitem['long'];
                        break;
                    }
                }
            }
        }

        stomp_unsubscribe($link, $resultqueue);
        stomp_close($link);
    }

    return $num;
}

$num = getMQStatus('spider');
```

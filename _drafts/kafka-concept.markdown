---
layout: post
title:  "kafka基本概念"
date:   2016-07-19 16:33:49
categories: java
tags: java
---

1.介绍
    kafka是一个高性能，分布式的消息队列，从高层视角来看，消息生产者（producer）通过网络把消息按顺序发送到kafka集群，然后消费者从集群中顺序读取，整体外观如下图：![外观图](/imgs/kafka1.png)

2.基本概念  

  * Topics：是某类消息的集合，可以包含多个分区（partition）

  * Partitions：一个topic可以包含一个或多个分区，多个分区可以位于集群的多个机器上，如果有多个分区，可以通过负载的方式（如轮询）把消息写入到分区中，可以  
  提高集群的性能。

  * Producers(生产者)：消息的产生者，产生消息，然后把消息发送到指定的topic中的分区中。

  * Consumers(消费者)：

[jekyll]:      http://jekyllrb.com
[jekyll-gh]:   https://github.com/jekyll/jekyll
[jekyll-help]: https://github.com/jekyll/jekyll-help

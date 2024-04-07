---
title: pika的codis集群方案【草稿】
author: june
date: 2024-04-01
category: db
layout: post
---
![codispika_f1](/assets/post/db/codispika_f1.png "codispika_f1")

pika的codis集群搭建需考虑的一些问题：
- 数据如何迁移：pika官方提供的两个工具pika-port，pika-migrate。
- codis新增group，slot如何迁移：codis提供的rebalance功能，或者手动指定新group的slot范围。
命令格式如下：`./codis-admin -v --dashboard=127.0.0.1:18080 --slot-action --create-range --beg=0 --end=3 --gid=1 --tid=0`
- 单个group pika中的高可用问题：codis的高可用是使用sentinel来实现的。但pika因为加了table之后，导致不再支持sentinel,所以这个没有办法再使用了。
目前是手工维护主从关系的。
- codis的监控：对codis-proxy进行二次开发，使其支持普罗米修斯，从而对操作命令qps，耗时等做监控。以及codis-proxy部署k8s集群上，k8s自身就对服务的pod资源使用有监控告警。


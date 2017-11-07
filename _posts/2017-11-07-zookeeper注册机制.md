---
layout:     post
title:      Zookeeper监听机制
subtitle:   详解zk监听流程
date:       2017-11-07
author:     刘旭刚
header-img: img/post-bg-2015.jpg
catalog: true
tags:
    - zookeeper
    - 分布式
---

# Watch简介

zookeeper提供了分布式的发布/订阅功能

# Watch流程

由客户端注册到zkManager 通过网络传输packet 服务端FinalReduestProcessor接收到服务的的注册请求，注册serverCnxn到WatchManager。同时客户端收到注册的结果，将本身业务注册到ZkWatchManager中。
# Watch源码实现




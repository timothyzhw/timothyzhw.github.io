---
layout:     post
title:      "CentOS虚拟机搭建"
subtitle:   ""
date:       2016-12-16
author:     "Tim zhong"
header-img: "/img/post/12-16-centos/bk.jpg"
catalog: true
tags:
    - centos
    - linux
---

> 最近要用Linux做测试，需要准备测试环境，将vmvare上CentOS的搭建过程记录一下

#  安装zookeeper

zookeeper下载，当前版本3.4.9。 上传到

tar zxvf  zookeeper-3.4.9
ln -s zookeeper-3.4.9 zookeeper
cp zoo_sample.cfg  zoo.cfg

设置

``` consle
tickTime=2000
dataDir=/var/lib/zookeeper
clientPort=2181
```
其中

* tickTime是ZooKeeper的基本时间单位（毫秒）。用来心跳检测，会话超时是2*tickTime.
* dataDir 是内存的数据库快照
* clientPort 客户端连接的端口


server.1=192.168.153.100:3333:4444  
server.2=192.168.153.101:3333:4444  
server.3=Slave2:3333:4444  


./zkServer.sh status
JMX enabled by default  
Using config: /home/hadoop/zookeeper-3.4.6/bin/../conf/zoo.cfg  
Mode: follower  
./zkServer.sh start-foreground 可以看到错误信息

三台

peerType=observer

server.1=10.20.0.4:13331:14441
server.2=10.20.0.4:13332:14442
server.3=10.20.0.4:13333:14443
server.4=10.11.0.4:13334:14444:observer
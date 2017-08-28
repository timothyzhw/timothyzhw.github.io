---
layout:     post
title:      "Elastic Stack 搭建日志查询"
subtitle:   ""
date:       2017-01-16
author:     "Tim zhong"
header-img: "/img/post/17-01-16-elastic/bk.svg"
catalog: true
tags:
    - elastic
    - linux
    - log
---

> The Open Source Elastic Stack:Reliably and securely take data from any source, in any format, and search, analyze, and visualize it in real time.

日志查询分析系统，可以监控生产环境的运行状态，快速高效排查线上问题。

面对大量的服务器，有效的方法是每台服务器上报日志，集中汇总到一起，由日志系统分析和查询日志。

比较流行的方案是使用 Elastic 栈的产品。

# Filebeat 安装

[Filebeat](https://www.elastic.co/products/beats/filebeat)是安装在服务器上的Agent，用来收集每台服务器的日志文件，并传送到Elasticsearch。

Filebeat运行的示意图：
(/img/post/17-01-16-elastic/filebeat.png)


## windows服务器的安装方法

压缩文件解压后，修改文件夹的名字为__Filebeat__

修改配置文件 __filebeat.yml__ 
1. 设置扫描log文件存放的位置和规则

windows服务器需要采集IISlog和应用的log，配置相应的log地址和规则。地址配置不支持递归查找，只能设置固定层级。
如c:\log\*\*.log，是采集c:\log\ 子目录中的.log文件，既不会采集c:\log\文件夹下的.log，也不会采集c:\log\log1\log2文件夹的文件。

```
filebeat.prospectors:
- input_type: log
  paths:
    - c:\vashare_log\*\*.txt
  document_type: applog
  multiline.pattern: '^\['
  multiline.negate: true
  multiline.match: after

- input_type: log
  paths:
    - C:\inetpub\logs\LogFiles\*\*.log
  document_type: iislog

```

2. 设置输出的Elasticsearch的IP和端口

```
output.elasticsearch:
  hosts: ["192.168.1.42:9200"]

```

然后运行Powershell脚本 _install-service-filebeat.ps1_，将Filebeat安装为Widnows服务。
ps1文件设置了默认的日志文件存放在c:\programdata\filebeat,可根据需要更改位置。

# Elasticsearch 安装

## 安装java 

从Oracle下载java jre, 并安装
sudo yum install lrzsz
sudo rz //上传安装文件

加压到/usr/lib/jvm文件夹

sudo vi /etc/profile
sudo vi /etc/bashrc

```
export JAVA_HOME=/usr/local/java
export PATH=$PATH:$JAVA_HOME/bin
export CLASSPATH=.:$JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar
```
source /etc/profile

./bin/elasticsearch -d -p pid

kill `cat pid`

使用sudo或者root运行elasticsearch，提示不能用root用户运行。普通用户提示权限问题。
chomd 777 -R *  

## Ingest Node

Ingest默认是开启的。


# Kibana 安装

下载 wget https://artifacts.elastic.co/downloads/kibana/kibana-5.1.2-linux-x86_64.tar.gz

tar -xzf kibana-5.1.2-linux-x86_64.tar.gz
ln -s kibana-5.1.2-linux-x86_64 kibana
cd kibana/
./bin/kibana

ps -ef | grep '.*node/bin/node.*src/cli'

#设置Pipeline

pipeline的ID是 __pl-app-log__

PUT _ingest/pipeline/pl-app-log
{
  "description" : "parse message",
  "processors" : [
    {
      "grok": {
        "field": "message",
        "patterns": ["\\[%{LOGLEVEL:level}\\] %{VATIME:date} %{USER:modul}\\.%{VATHREAD:thread}: %{VAINFO:info}"],
        "pattern_definitions":{
            "VATIME" : "\\d\\d\\.\\d\\d-\\d\\d:\\d\\d:\\d\\d.\\d*",
            "VATHREAD" : "T\\d*",
            "VAINFO" : "[\\s\\S]*"
        }
      }
    }
  ]
}

PUT _ingest/pipeline/app-pipeline
{
  "description" : "parse app message",
  "processors" : [
    {
      "grok": {
        "field": "message",
         "ignore_failure" : false,
        "patterns": ["\\[%{LOGLEVEL:level}\\] %{VATIME:date} %{USER:modul}\\.%{VATHREAD:thread}: %{VAINFO:info}"],
        "pattern_definitions":{
            "VATIME" : "\\d\\d\\.\\d\\d-\\d\\d:\\d\\d:\\d\\d.\\d*",
            "VATHREAD" : "T\\d*",
            "VAINFO" : "[\\s\\S]*"
        }
      }
    }
  ],
    "on_failure" : [
    {
      "set" : {
        "field" : "_index",
        "value" : "failed-applog"
      }
    }]
}

 PUT _ingest/pipeline/iis-pipeline
{
    "description": "parse iis message",
    "processors": [
      {
        "grok": {
          "field": "message",
          "ignore_failure" : true,
          "patterns": [
            "%{TIMESTAMP_ISO8601:log_timestamp} %{IPORHOST:site} %{WORD:method} %{NOTSPACE:page} %{NOTSPACE:querystring} %{NUMBER:port} %{NOTSPACE:username} %{IPORHOST:clienthost} %{NOTSPACE:useragent} %{NOTSPACE:referer} %{NOTSPACE:host} %{NUMBER:response} %{NUMBER:subresponse} %{NUMBER:scstatus} %{NUMBER:time_taken}"
          ]
        }
      }
    ],
    "on_failure" : [
    {
      "set" : {
        "field" : "_index",
        "value" : "failed-iislog"
      }
    }]
  }

 PUT _ingest/pipeline/statistics-pipeline
 {
    "description": "parse json and convert date to @timestamp",
  "processors":[
      {
        "json": {
          "field": "message",
          "add_to_root": true
        }
      },
      {
        "remove": {
          "field": "message"
        }
      },
      {
        "date": {
          "field": "Date",
          "formats": [
            "yyyy-MM-dd'T'HH:mm:ss.SSSZ"
          ],
          "target_field": "@timestamp"
        }
      },
      {
        "remove": {
          "field": "Date"
        }
      },
      {
        "date_index_name": {
          "field": "@timestamp",
          "index_name_prefix": "statistics-",
          "date_rounding": "d"
        }
      }
    ],
     "on_failure" : [
    {
      "set" : {
        "field" : "_index",
        "value" : "failed-statistics"
      }
    }]
  }

POST _ingest/pipeline/iis-pipeline/_simulate
{
  "docs" : [
    {
      "_source": {"message":"2017-01-23 02:47:50 172.16.4.5 GET /account/Vacation - 80 - 220.249.11.254 Mozilla/5.0+(Windows+NT+10.0;+WOW64)+AppleWebKit/537.36+(KHTML,+like+Gecko)+Chrome/55.0.2883.87+Safari/537.36 http://www.vashare.com/account/vacation/ 200 0 0 159"}
    },
    {
      "_source":
       {"message":"2017-01-23 02:47:50 172.16.4.5 GET /favicon.ico - 80 - 220.249.11.254 Mozilla/5.0+(Windows+NT+10.0;+WOW64)+AppleWebKit/537.36+(KHTML,+like+Gecko)+Chrome/55.0.2883.87+Safari/537.36 http://www.vashare.com/account/Vacation 200 0 0 4"}
    }
  ]
}

获取pipeline
GET  _ingest/pipeline
GET  _ingest/pipeline/pipeline-name

获取index

# Metricbeat
(https://www.elastic.co/downloads/beats/metricbeat)

## YUM 安装

1. 安装YUM的公共签名

sudo rpm --import https://packages.elastic.co/GPG-KEY-elasticsearch

2. 在/etc/yum.repos.d/ 文件夹下新建elastic.repo，并且输入下面的内容：

[elastic-5.x]
name=Elastic repository for 5.x packages
baseurl=https://artifacts.elastic.co/packages/5.x/yum
gpgcheck=1
gpgkey=https://artifacts.elastic.co/GPG-KEY-elasticsearch
enabled=1
autorefresh=1
type=rpm-md

sudo yum install metricbeat

3. 修改配置文件

文件位于/etc/metricbeat

```
#-------------------------------- Redis Module -------------------------------
- module: redis
  metricsets: ["info", "keyspace"]
  enabled: true
  period: 10s

  # Redis hosts
  hosts: ["127.0.0.1:63379","127.0.0.1:63380","127.0.0.1:63381"]

  # Enabled defines if the module is enabled. Default: true
  #enabled: true

  # Timeout after which time a metricset should return an error
  # Timeout is by default defined as period, as a fetch of a metricset
  # should never take longer then period, as otherwise calls can pile up.
  timeout: 1s

  # Optional fields to be added to each event
  fields:
    datacenter: zh

  # Network type to be used for redis connection. Default: tcp
  #network: tcp

  # Max number of concurrent connections. Default: 10
  #maxconn: 10

  # Filters can be used to reduce the number of fields sent.
  #filters:
  #  - include_fields:
  #      fields: ["stats"]

  # Redis AUTH password. Empty by default.
  #password: foobared
```

3. 启动

sudo /etc/init.d/metricbeat start

curl -XGET 'http://localhost:9200/metricbeat-*/_search?pretty'

# 处理Pipeline的异常

在管道处理中出现异常时，可以在pipeline或 processor中设置 on_failure 参数。
当

 

 
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

# 更换yum源

为了加速yum安装的速度，更换yum源为阿里云，下载速度快很多。

{%highlight shell %}

#备份
mv /etc/yum.repos.d/CentOS-Base.repo /etc/yum.repos.d/CentOS-Base.repo.backup
#下载新的CentOS-Base.repo 到/etc/yum.repos.d/
wget -O /etc/yum.repos.d/CentOS-Base.repo http://mirrors.aliyun.com/repo/Centos-6.repo

{%endhighlight%}

*修改其他版本CentOS源参见 [阿里云](http://mirrors.aliyun.com/help/centos) 的帮助页。

# 安装 lrzsz

lrzsz是一款在linux里可代替ftp上传和下载的程序。安装后可以往CentOS上传和下载文件。

{%highlight shell %}
sudo yum install lrzsz
sudo rz    #上传安装文件，弹出的窗口里选择要上传的文件
{%endhighlight%}


service iptables stop #停止
chkconfig iptables off #禁用







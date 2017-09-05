---
layout:     post
title:      "Docker 实战"
subtitle:   ""
date:       2017-08-28
author:     "Tim zhong"
header-img: "/img/post/17-01-16-elastic/bk.svg"
catalog: true
tags:
    - docker
    - linux
---

# Dockerfile

Dockerfile定义了容器内的环境配置，如网络端口映射和虚拟磁盘等。

示例：

```bash
# Use an official Python runtime as a parent image
FROM python:2.7-slim

# Set the working directory to /app
WORKDIR /app

# Copy the current directory contents into the container at /app
ADD . /app

# Install any needed packages specified in requirements.txt
RUN pip install -r requirements.txt

# Make port 80 available to the world outside this container
EXPOSE 80

# Define environment variable
ENV NAME World

# Run app.py when the container launches
CMD ["python", "app.py"]

```

# requirements.txt

```
Flask
Redis
```

# app.py

```python

from flask import Flask
from redis import Redis, RedisError
import os
import socket

# Connect to Redis
redis = Redis(host="redis", db=0, socket_connect_timeout=2, socket_timeout=2)

app = Flask(__name__)

@app.route("/")
def hello():
    try:
        visits = redis.incr("counter")
    except RedisError:
        visits = "<i>cannot connect to Redis, counter disabled</i>"

    html = "<h3>Hello {name}!</h3>" \
           "<b>Hostname:</b> {hostname}<br/>" \
           "<b>Visits:</b> {visits}"
    return html.format(name=os.getenv("NAME", "world"), hostname=socket.gethostname(), visits=visits)

if __name__ == "__main__":
    app.run(host='0.0.0.0', port=80)
```


# 命令集合

Here is a list of the basic Docker commands from this page, and some related ones if you’d like to explore a bit before moving on.

```bash
docker build -t friendlyname .  # Create image using this directory's Dockerfile
docker run -p 4000:80 friendlyname  # Run "friendlyname" mapping port 4000 to 80
docker run -d -p 4000:80 friendlyname         # Same thing, but in detached mode
docker container ls                                # List all running containers
docker container ls -a             # List all containers, even those not running
docker container stop <hash>           # Gracefully stop the specified container
docker container kill <hash>         # Force shutdown of the specified container
docker container rm <hash>        # Remove specified container from this machine
docker container rm $(docker container ls -a -q)         # Remove all containers
docker image ls -a                             # List all images on this machine
docker image rm <image id>            # Remove specified image from this machine
docker image rm $(docker image ls -a -q)   # Remove all images from this machine
docker login             # Log in this CLI session using your Docker credentials
docker tag <image> username/repository:tag  # Tag <image> for upload to registry
docker push username/repository:tag            # Upload tagged image to registry
docker run username/repository:tag                   # Run image from a registry
```

# Services

服务是分布式环境下的各种app。服务是生产环境下的容器。一个服务一个镜像image，并包括镜像运行的方法，包括端口、镜像数量。还有缩放的数量。使用yml文件来定义。

## 第一个 docker-compose.yml 文件

```yaml
version: "3"
services:
  web:
    # replace username/repo:tag with your name and image details
    image: username/repository:tag
    deploy:
      replicas: 5
      resources:
        limits:
          cpus: "0.1"
          memory: 50M
      restart_policy:
        condition: on-failure
    ports:
      - "80:80"
    networks:
      - webnet
networks:
  webnet:

```

```bash
docker stack ls              # List all running applications on this Docker host
docker stack deploy -c <composefile> <appname>  # Run the specified Compose file
docker stack services <appname>       # List the services associated with an app
docker stack ps <appname>   # List the running containers associated with an app
docker stack rm <appname>                             # Tear down an application
```

# 网络

Docker通过网络驱动器来支持容器的网络。默认提供两个网络驱动： ___bridge___ 和 ___overlay___  。 

安装docker后自动带了三个网络

```
$ docker network ls

NETWORK ID          NAME                DRIVER
18a2866682b8        none                null
c288470c46f6        host                host
7b369448dccb        bridge              bridge
```

默认的容器是在bridge网络下。

![Bridge网络](/img/post/18-08-28-docker/bridge1.png)

可以使用命令来查看容器的ip地址

```
$ docker network inspect bridge
```

同样可以使用命令将容器从网络移除

```
$ docker network disconnect bridge networktest
```

## 创建自己的bridge网络

```
$ docker network create -d bridge my_bridge
```

参数-d表示使用bridge驱动。

使用ls查看新建的网络，使用inspect检查网络，没有容易在该网络下。

通过定义网络可以隔离出独立的容器环境。在建立容器时，使用 --net=my_bridge 参数来指定容器的网络。
```
$ docker run -d --net=my_bridge --name db training/postgres

$ docker run -d --name web training/webapp python app.py
```
在my_bridge网络下建立db，在默认网络下建立web

![图片](/img/post/18-08-28-docker/bridge2.png)

打开容器db的shell，运行ping命令，ping不通web

```
$ docker exec -it db bash

$ exit
```

但是docker允许在容器上附加多个网络。下面就将my_bridge附加到web容器上，这时候web和db都处于一个网络中了。

```
$ docker network connect my_bridge web
```

![图片](/img/post/18-08-28-docker/bridge3.png)


docker inspect --format='{{json .NetworkSettings.Networks}}'  web

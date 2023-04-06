---
layout:     post
title:      docker问题
subtitle:   docker问题
date:       2023-04-06
author:     xc
header-img: img/post-bg-ios9-web.jpg
catalog: true
tags:
    - OS
    - linux
---
# docker中ubuntu系统没有任何命令
```
sed -i s@/archive.ubuntu.com/@/mirrors.aliyun.com/@g /etc/apt/sources.list

sed -i s@/security.ubuntu.com/@/mirrors.aliyun.com/@g /etc/apt/sources.list

apt-get clean

apt-get update

apt install net-tools

apt-get install vim

apt-get -y install netcat-traditional

update-alternatives --config nc
```
上述命令用于更新apt

# 系统中没有python
```
apt install python3
```

# 给docker分配固定ip

1.搭建网卡

docker network create --subnet=192.168.1.0/24 --gateway=192.168.1.1 docker-intranet

2.指定网卡和IP

sudo docker run -it --name=HostA --hostname=HostA --net=intranet --ip=192.168.60.2 --privileged "seedubuntu" /bin/bash


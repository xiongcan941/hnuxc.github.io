---
layout:     post
title:      sqlmap --os-shell原理
subtitle:   sqlmap --os-shell原理
date:       2023-04-15
author:     xc
header-img: img/post-bg-ios9-web.jpg
catalog: true
tags:
    - web安全
---

# sqlmap --os-shell原理
必要条件：
拥有网站的写入权限
secure_file_priv参数为空或者为指定路径
知道网站绝对路径
就是上传一个php的马到那里，一个用来上传文件，一个用上一个上传文件的马来上传可以执行命令的木马退出时删除Shell

对于数据库
数据库必须支持外联
上传一个二进制库，包含用户自定义的函数
连接mysql数据库并且获取数据库版本
检测是否为数据库dba
检测sys_exec和sys_eval2个函数是否已经被创建
上传dll文件到对应目录
用户退出时删除两个函数
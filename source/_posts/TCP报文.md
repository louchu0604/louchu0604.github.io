---
layout: _post
title: TCP报文
date: 2019-06-03 23:45:57
tags:
---
先看一下TCP报文结构

![TCP报文结构](https://cy-resource.oss-cn-shanghai.aliyuncs.com/blogimage/TCP%E6%8A%A5%E6%96%87%E7%BB%93%E6%9E%84.jpg "TCP报文结构")
<!-- 插入图片 -->
TCP头部结构
![TCP头部结构](https://cy-resource.oss-cn-shanghai.aliyuncs.com/blogimage/TCP%E5%A4%B4%E9%83%A8%E7%BB%93%E6%9E%84.jpg "TCP报文结构")


<!-- 插入图片 -->
每个字段的含义 
`CWR`:
`ECE`:
`URG`:
`ACK`:确认号
`PSH`:
`RST`:
`SYN`:初始化序列号
`FIN`:
以上内容解释完之后，来看一下TCP为什么是`安全可靠`的
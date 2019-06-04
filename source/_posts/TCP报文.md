---
layout: _post
title: TCP报文
date: 2019-06-03 23:45:57
tags:
---
这是读《TCP/IP》的笔记之一，很多内容都是书上看的，然后图片是根据书中的内容绘制出来的。
ps：TCP/IP这本书，真的有点厚，不知道什么时候能看完。
先看一下TCP报文结构

![TCP报文结构](https://cy-resource.oss-cn-shanghai.aliyuncs.com/blogimage/TCP%E6%8A%A5%E6%96%87%E7%BB%93%E6%9E%84.jpg "TCP报文结构")
<!-- 插入图片 -->
TCP头部结构
![TCP头部结构](https://cy-resource.oss-cn-shanghai.aliyuncs.com/blogimage/TCP%E5%A4%B4%E9%83%A8%E7%BB%93%E6%9E%84.jpg "TCP报文结构")


<!-- 插入图片 -->
每个字段的含义 
`CWR`:通知接收方，已将拥塞窗口缩小（跟下面的ECE一起用）
`ECE`:通知接收方，接收方到这边的网络有阻塞
`URG`:头部的紧急指针是否有效
`ACK`:响应
`PSH`:不缓存，马上处理
`RST`:连接重置
`SYN`:建立连接
`FIN`:关闭连接
以上内容解释完之后，来看一下TCP为什么是`安全可靠`的

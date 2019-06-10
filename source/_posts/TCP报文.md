---
layout: _post
title: TCP连接管理与保活机制
date: 2019-06-03 23:45:57
tags: TCP
---
这是读《TCP/IP》的笔记之一，很多内容都是书上看的，然后图片是根据书中的内容绘制出来的。
ps：TCP/IP这本书，真的有点厚，不知道什么时候能看完。
# TCP报文结构

![TCP报文结构](http://static.kaolagogogo.fun/blogimage/TCP%E6%8A%A5%E6%96%87%E7%BB%93%E6%9E%84.jpg "TCP报文结构")
<!-- 插入图片 -->
TCP头部结构
![TCP头部结构](http://static.kaolagogogo.fun/blogimage/TCP%E5%A4%B4%E9%83%A8%E7%BB%93%E6%9E%84.jpg "TCP报文结构")


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

# TCP连接管理
以上内容解释完之后，来看一下TCP为什么是`安全可靠`的

首先是连接的过程，采用了我们耳熟能详的“三次握手”，
请看下面这张图片
![三次握手](
http://static.kaolagogogo.fun/blogimage/TCP%E4%B8%89%E6%AC%A1%E6%8F%A1%E6%89%8B.jpg "三次握手")
解释一下其中的几个状态: 
* SYN-SENT:客户端在发送一个连接请求之后，收到应答之前的状态
* SYN-RECEIVED (SYN_RCVD)：服务端在接收到一个连接请求和发送一个链接请求之后，收到应答之前的状态
第一次和第二次握手可以保证客户端向服务端发送数据是可靠的
第二次和第三次握手可以保证服务端向客户端发送数据是可靠的
* 第一次的序列号为什么是随机的
    * 防止同一个连接的不同实例
    * 防止TCP系列号欺骗
    * 随机序列号的生成： ISN = M + F(localhost, localport, remotehost, remoteport)

断开连接过程，采用的是“四次挥手”
![四次挥手](http://static.kaolagogogo.fun/blogimage/TCP%E5%9B%9B%E6%AC%A1%E6%8C%A5%E6%89%8B.jpg "四次挥手")
解释一下其中的几个状态
* FIN_WAIT_1  客户端发出断开连接的请求后，等待ack回复的状态 
* CLOSE_WAIT  服务端等待对方的断开连接请求 
* FIN_WAIT_2  等到服务端的断开请求
* LAST_ASK 服务端发出FIN后 等待客户端的响应
* TIME_WAIT 客户端收到服务端的FIN 发出ACK之后 等待一段时间（防止这个ACK包没有成功发送）


 # 保活
 ## 过程
发送端开启保活功能
1. 在保活时间内如果处于非活动状态，发送保活探测报文
2. 如果没有响应，则经过保活时间间隔之后继续发送，直到达到上限（这里的上限指的是保活探测数）
3. 如果没有响应，则认为数据无法到达，连接将被断开
4. 保活探测报文为空报文或一个字节的报文，序列号为接收端发送的ACK报文的最大序列号减1（原因：这个序列号接收端已经收到 所以不会影响之前的数据）

## 结果
1. 接收端仍在工作，并响应了探测报文
2. 接收端已断开
3. 接收端重启了
4. 接收端仍在工作，但因为某种原因没有收到ACK

#UDP的连接 
#TCP fastopen（TFO）
TCP一般情况下需要三次握手才进行数据传输，相形之下，比UDP多了数据传输时间
TFO就是在这样的背景下提出来的，它允许TCP在握手阶段就传输数据，从而节省时间。
## TFO的过程
### 客户端通过一个普通的三次握手获取FOC（fast open cookie）
* 客户端发送一个带有fast open选项的的SYN ，同时携带一个空的cookie域来请求cookie
* 服务端收到后，产生一个cookie，通过SYN-ACK包的fast open选项返回
* 客户端缓存该cookie
### 执行TFO 
* 客户端发送带有数据的SYN包，同时在fast open选项中携带之前的cookie
* 服务端收到后，验证该cookie，如果cookie有效，服务端返回SYN-ACK包，同时把接收到的数据传递给应用层。如果cookie无效，服务端会丢弃该包中的数据，同时返回一个SYN-ACk
* cookie如果有效，服务端可以发送相应数据
* 客户端发送ACK来确认服务端的SYN和数据，如果客户端的数据没有被确认（cookie失效之类的），客户端将重传数据（重传机制）
* 客户端获取到FOC之后，可以重复fast open 直到cookie过去
Q1：服务端如何验证cookie

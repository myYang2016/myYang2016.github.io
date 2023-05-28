---
title: websocket学习
date: 2022-03-08 15:16:26
tags:
---
### websocket学习
> 系统地学习websocket

#### 什么是websocket

现代web应用中，实现信息交互的需求比较复杂，需要全双工通信。而websocket协议就是针对这点，适合文本信息的双向通信的。
而传统的http，是单向的。而可以利用轮询、长链接等手段，也能在http下实现双向通信，但是由于http流程繁琐，且每次请求携带的头部信息比较多，导致贷款和延时比较高，性能低。
除了websocket，实时双向通信还有SSE、SPDY、webrtc等。SSE主要用来服务端单向通知客户端信息的。SPDY是在文档信息实时通信应用的。WEBRTC是音视频实时通信的。

#### websocket api
可以使用on<event>或者addEventListen，跟js原生的事件机制是一样的。
事件有open、disconnect、error、close
有websocket.readyState表示当前socket的链接状态，0表示正在链接中，1表示链接成功。
可以传送字符串和二进制类型，其中可以跟File和webrtc等api结合使用。

#### websocket 协议
websocket是在http协议的基础上，加上upgrade：websocket，来对协议进行升级成为websocket连接。
连接需要发送响应键值。客户端连接是，通过set-websocket-key来传递键值，服务端经过添加协议串并hash后，通过字段set-websocket-Accept返回。
websocket本身可以关闭，但也有各种异常关闭。

#### 安全
每次websocket握手连接时，携带的set-websocket-key以及服务端响应的set-websocket-Accept都是为了保护非websocket服务器不会收到跨协议攻击。因为如果没有set-websocket-Accept，非websocket服务器会被当成websocket使用，从而造成跨协议攻击。屏蔽主要是在传递的websocket消息中，对二进制进行异或操作，从而起到安全作用。


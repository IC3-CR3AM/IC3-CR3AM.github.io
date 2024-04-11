---
title: frp源码分析小记
date: 2024-04-11 19:46:50
tags:
---

# frp源码分析小记

翻看很多文章对frp的运行原理和实现相当模糊，于是萌生了阅读源码的想法，以下是阅读代码后整理的一些笔记

目前最新的dev版本在0.56

### frp基础结构

最新版的代码结构非常清晰易懂，其主体结构自cmd始，分别在client和server中实现，最后封装了底层各种库，看不到复杂的细节。

这里先仅仅看client↔server的交互，作用是作为整个链路建立的底层，相当于会话层和表示层

在正常工作下，整个交互过程如下：

1. client轮询登陆server
2. server验证并返回属于该client的RunID，此时client收到数据后会立即断开login的conn
3. client会根据心跳时间发起下一个login请求，此时第二个才是真正负责起心跳的信道。如果这条信道断了，快速重发三次，间隔200毫秒，再是20秒的间隔重发
4. server在新线程里向client发送一个`msg.NewWorkConn`，请求workConn（真正工作的连接）
5. client收到请求后向server发起workConn
6. server接收到请求，注册建立起的请求，并放入资源池

### 协议

接下来的流程根据用户设定的协议来

**stcp场景：**

需要用户自己在本机也是用frpc，称之为visitor。

1. 本机上根据配置文件，需要开端口的打开端口等待连接
2. 如果用户有连接访问该端口，该frpc会发起`msg.NewVisitorConn`请求，
    1. client会发起`msg.NewProxy`请求，让server创建内部监听器。
3. server验证visitor，通过后
    1. server，对通过的conn中的数据进行流式加密和压缩（这些都是作者手搓的），最后将该conn加入内部队列，proxy的internalListener.Accept获取到该conn
    2. 同时本机，frpc会对传入conn和本机至server的conn进行数据交换
4. server的proxy从workpool（也就是资源池）中寻找存在的workConn，如果没有就向client申请一条，最后交换workConn和visitor的conn流量
5. 目标机client的proxy_manager处理之前`msg.NewWorkConn`请求的返回，会根据返回数据，向对应的服务发起请求，再把这个服务建立的连接与workConn进行数据交换

这样一套完整的连接，其中要发生三次数据交换，frp的代码为`libio.Join(localConn, remote)`

**反向socks5场景：**

1. client根据配置文件，发起带有端口的`msg.NewProxy`请求
2. server收到请求，运行tcp代理并打开对应端口监听
3. 用户使用socks5连接server打开的端口，server的proxy从workpool中寻找workConn，如果没有就向client申请一条，最后交换workConn和用户conn的流量
4. 最后，client处理返回的workConn时要打开并解析该conn，即`armon/go-socks5`实现的解析协议。根据用户配置，创建新的连接，访问内网目标。

### 小结

本文对frp部分的协议进行了分析，其代码实现有很多精彩的地方，为了清晰易懂并为将其放入，强烈推荐由相关需求的同学读一遍源码。尤其它的结构是它的闪光点，但加解密部分还需要打磨。
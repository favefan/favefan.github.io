---
title: "陈硕的网络编程实践 1-9 概览与TTCP"
date: 2024-01-20T20:20:20
description: "听陈硕的网络编程实践的记录, 1-9节视频, 主要介绍了课程的概要以及TTCP的代码和原理, 还有一些TCP的坑."
categories: Network_Programming
tags: [C/C++, Network_Programming, Chen_Shuo]
hidden: false
---

## 01 网络编程概要

### 分层的网络

* 以太网 Ethernet 帧 Frame
* IP 分组 Packet
* TCP 分节 Segment
* 应用层 消息 Message

### 初学者的错误

* 避免将网络编程 (Network I/O) 与业务逻辑混在一起, 不能良好的可持续开发
* 收到的 TCP 数据不完整 (不可靠): TCP 连接断开的时机与条件, close 太早可能导致协议栈发送 RST 分节, 将连接重置
* TCP 分包逻辑中的坑
* 应该避免直接发送 C 中的结构体
    * 要考虑对齐问题
    * 高度不可扩展 (如一端修改协议, 其他节点均需要修改)
* TCP 的自连接: TCP 客户端在与本机的 TCP 服务器建立连接时, 服务器未正常启动, 则有可能客户端自己与自己连接
* 非阻塞网络编程

## 02 一个 TCP 的简单实验

## 03 课程内容大纲

### 基本的非并发例子 non-concurrent

* TTCP: 经典的 TCP 性能测试工具
* Round-trip: 测试两台主机之间的时间差 (唯一的 UDP 例子)
* Netcat: 网络编程的瑞士军刀, 与标准输入输出相关的例子
* Slow sink/source: 慢速收发, 模拟慢网络环境的通信, 从应用层模拟

### 并发网络编程

* SOCKS协议 代理服务器
* 数独求解器
    * 典型的请求-响应式 (request-response) 模型
* 简单的 memcached
* 应用层的 TCP 广播

### 用多台机器进行数据处理

* 并行的 N 皇后问题求解
* 求分布在多台机器上的数据的中位数
* Frequent queries
* 分布式排序 - MapReduce 的难点之一

### 高级主题

* RPC
* 负载均衡
* 服务器系统的容量管理
* ...

## 04 回顾基础的 Sockets (TTCP)

### 我们关心的性能指标

* 带宽 Bandwidth MB/s
* 吞吐量 Throughout (应用层面): message/s, queries/s (QPS), transactions/s (TPS)
* 延迟 Latency millisecond, 百分位数的延迟 50%, 95%, 99%, ...
* 资源使用率 Utilization percent 百分比
* 额外开销 Overhead, CPU 利用率: 压缩, 加密. 减少额外开销所耗费的时间: 边压缩边传送

开发中进行性能估算来考虑优化的方法与效果, 必要性

### 为什么使用 TTCP

* 使用了基本的 Sockets API: socket, listen, bind, accept, connect, read/recv, write/send, shutdown, close, etc.
* 协议有格式, 不是单纯的字节流, 比 echo 要好
* 具有 TCP 的典型行为, 可以与实际程序中的方法行为进行比较来衡量性能
* 用多种语言均能实现, 反应不同语言的性能
* 无并发

### TTCP 协议详解

![ttcp_protocol](https://s3.bmp.ovh/imgs/2024/01/19/711d7ac04e8f9cf2.png)

收到 ACK 后才会发送下一个 PayloadMessage.

协议消息定义:

```c
/* in common.h, total 8 bytes */
struct SessionMessage {
    int32_t number;
    int32_t length;
} __attribute__ ((__packed__));

/* in common.h, total 4+sizeof(data) bytes */
struct PayloadMessage {
    int32_t length;
    char data[0]; // 不定长数组, 长度在运行时决定
};
```

## 05 TTCP 代码概览

使用 c 与 sockets 的阻塞代码示例 (收与应答):

*in muduo/examples/ace/ttcp/ttcp_blocking.cc*

```c
void receive(const Options& opt)
{
  /* 接受一个连接 */
  int sockfd = acceptOrDie(opt.port); 

  /* in common.h, totally 8 bytes
  struct SessionMessage
  {
    int32_t number;
    int32_t length;
  } __attribute__ ((__packed__)); 
  */
  struct SessionMessage sessionMessage = { 0, 0 };
  if (read_n(sockfd, &sessionMessage, sizeof(sessionMessage)) != sizeof(sessionMessage))
  {
    perror("read SessionMessage");
    exit(1);
  }

  /* 网络字节序转为本机字节序 */
  sessionMessage.number = ntohl(sessionMessage.number);
  sessionMessage.length = ntohl(sessionMessage.length);
  printf("receive number = %d\nreceive length = %d\n",
         sessionMessage.number, sessionMessage.length);
  /* 计算接收大小 */
  const int total_len = static_cast<int>(sizeof(int32_t) + sessionMessage.length);
  /* 安全漏洞: 如果 length 太大, 则 malloc 分配 "巨大" 的空间, 造成拒绝响应攻击 */
  /* 可以添加逻辑来限制 length 的大小 */

  /* in common.h
  struct PayloadMessage
  {
    int32_t length;
    char data[0]; // 不定长数组, 长度在运行时决定
  };
  */
  PayloadMessage* payload = static_cast<PayloadMessage*>(::malloc(total_len));
  assert(payload);

  for (int i = 0; i < sessionMessage.number; ++i)
  {
    payload->length = 0;
    /* 一收 */
    if (read_n(sockfd, &payload->length, sizeof(payload->length)) != sizeof(payload->length))
    {
      perror("read length");
      exit(1);
    }
    payload->length = ntohl(payload->length);
    assert(payload->length == sessionMessage.length);
    if (read_n(sockfd, payload->data, payload->length) != payload->length)
    {
      perror("read payload data");
      exit(1);
    }
    int32_t ack = htonl(payload->length);
    /* 一应答 */
    if (write_n(sockfd, &ack, sizeof(ack)) != sizeof(ack))
    {
      perror("write ack");
      exit(1);
    }
  }
  ::free(payload);
  ::close(sockfd);
}
```

使用 c++ 与 Wrapped sockets 的阻塞代码示例 (发送与测量):

*in recipes/tpc/ttcp.cc*

```c++
void transmit(const Options& opt)
{
  InetAddress addr(opt.port);
  if (!InetAddress::resolve(opt.host.c_str(), &addr))
  {
    printf("Unable to resolve %s\n", opt.host.c_str());
    return;
  }

  printf("connecting to %s\n", addr.toIpPort().c_str());
  /* in TcpStream.h
  typedef std::unique_ptr<TcpStream> TcpStreamPtr;
  // in TcpStream.cc
  TcpStream::TcpStream(Socket &&sock) : sock_(std::move(sock)) {} // && 右值引用
  // in C++ Primer 5th p471 13.6 对象移动 note: 标准库容器, string 和 shared_ptr 类既支持移动也支持拷贝. IO 类和 unique_ptr 类可以移动但不能拷贝
  */
  TcpStreamPtr stream(TcpStream::connect(addr)); // 使用了 c++11 的移动语义
  if (!stream)
  {
    printf("Unable to connect %s\n", addr.toIpPort().c_str());
    perror("");
    return;
  }

  if (opt.nodelay)
  {
    /* void Socket::setTcpNoDelay(bool on) {
        int optval = on ? 1 : 0;
        if (::setsockopt(sockfd_, IPPROTO_TCP, TCP_NODELAY,
                    &optval, static_cast<socklen_t>(sizeof optval)) < 0) {
            perror("Socket::setTcpNoDelay");
        }
    } */
    stream->setTcpNoDelay(true); // NO_NAGLE
  }
  printf("connected\n");
  double start = now();
  struct SessionMessage sessionMessage = { 0, 0 };
  sessionMessage.number = htonl(opt.number);
  sessionMessage.length = htonl(opt.length);
  if (stream->sendAll(&sessionMessage, sizeof(sessionMessage)) != sizeof(sessionMessage))
  {
    perror("write SessionMessage");
    return;
  }

  const int total_len = sizeof(int32_t) + opt.length;
  PayloadMessage* payload = static_cast<PayloadMessage*>(::malloc(total_len));
  std::unique_ptr<PayloadMessage, void (*)(void*)> freeIt(payload, ::free); // RAII
  assert(payload);
  payload->length = htonl(opt.length);
  for (int i = 0; i < opt.length; ++i)
  {
    payload->data[i] = "0123456789ABCDEF"[i % 16];
  }

  double total_mb = 1.0 * opt.length * opt.number / 1024 / 1024;
  printf("%.3f MiB in total\n", total_mb);

  for (int i = 0; i < opt.number; ++i)
  {
    /* 一发 */
    int nw = stream->sendAll(payload, total_len);
    assert(nw == total_len);

    int ack = 0;
    /* 一收 */
    int nr = stream->receiveAll(&ack, sizeof(ack));
    assert(nr == sizeof(ack));
    ack = ntohl(ack);
    assert(ack == opt.length);
  }

  double elapsed = now() - start;
  printf("%.3f seconds\n%.3f MiB/s\n", elapsed, total_mb / elapsed);
}
```

## 06 使用 TTCP 进行网络传输性能测试

## 07 阻塞 IO 下的TTCP实验 (echo)

*recipes/tpc/echo.cc & echi-client.cc*

### 阻塞 IO 会永远阻塞下去

![blocking_io_forever_blocking](https://s3.bmp.ovh/imgs/2024/01/19/f1c861a165506ca6.png)

因为内核设置的缓冲区大小不足以使得"及时" Recv 发送来的数据. 首先导致 Server 阻塞在 Send 上, 然后服务端不能 Read, 接连导致 Client 不能 Send. 究其原因客户端发送的数据太大, 且服务器准备的缓冲区太小.

阻塞 IO 在发生长期阻塞的情况下是无法自行自我解脱的.

举例的"20MB"的大小是合理合法的. 所以问题是 Server 的问题: 不能完整读到客户端发来的请求.

**Client 首先应该发送一个 header 告知服务器需要接受的请求的数量和大小, 然后再发送 payload 承载数据到服务器. Server 收到 header 后应该准备客户端需求大小的缓冲区 (如20MB), Server 发送数据到应用端前也应该先发送一个 header 告知客户端需要接受的数据情况, 这是在应用端需要考虑到的事情.**

字节流协议还是应该加上自己的分包措施.

## 08 TCP self-connection 自连接

*net.ipv4.ip_local_port_range*

*recipes/python/self-connect.py*

```python
#!/usr/bin/python

import errno
import socket
import sys
import time

if len(sys.argv) < 2:
    print "Usage: %s port" % sys.argv[0]
    print "port should in net.ipv4.ip_local_port_range"
else:
    port = int(sys.argv[1])
    for i in range(65536):
        try:
            sock = socket.create_connection(('localhost', port))
            print "connected", sock.getsockname(), sock.getpeername()
            time.sleep(60*60)
        except socket.error, e:
            if e.errno != errno.ECONNREFUSED:
                break
```

### 为什么会出现自连接

> 关于 TCP 同时打开 (from Github Copilot chat):
>
> * 在 Linux 中, TCP 同时打开 (TCP Simultaneous Open) 是指两个主机几乎同时尝试建立彼此的 TCP 连接. 这是一种罕见的情况, 但 TCP 协议确实为此提供了支持. 
>
> * 在 TCP 同时打开的情况下, 两个主机都会发送 SYN (同步) 包, 并且都会回应对方的 SYN 包. 这样, 两个主机都会进入 SYN-RECEIVED 状态. 然后, 当它们收到对方的 ACK (确认) 包时, 都会进入 ESTABLISHED 状态, 从而完成连接. 
>
> * 这种情况在实际网络环境中很少发生, 因为通常一个主机会先于另一个主机开始连接尝试. 然而, TCP 协议设计者考虑到了这种可能性, 并在协议中包含了处理这种情况的机制. 

* 当我们尝试连接同一主机上的某个套接字时, 如果我们没有将客户端绑定到特定端口, 操作系统会为您提供短暂端口, 如果该端口恰好是您要连接的端口, 则会导致自连接. 
* 加入检查逻辑来规避这种情况的发生:
    ```c++
    if (sock.connect.serverAddr == 0 && !isSelfConnection(sock)) {
        stream.reset(new TcpStream(std::move(sock)));
    }
    ...
    bool isSelfConnection(const Socket& sock) {
        return sock.getLocalAddr() == sock.getPeerAddr();
    }
    ```

## 09 TTCP 的扩展练习

* 测量吞吐量 throughout (message/s) 和延迟 latency (百分位数需要数组来辅助计算, 直方图)
* 服务器支持并发的客户端连接 (block: thread-per-connection, non-block: epoll or poll, etc.)
* 客户端支持多会话/连接, 测试 TCP 的公平性 (每个连接的资源相等)
* 物理上长距离的连接
  * Pipelining 流水线解决方法
    * ![tcp_Pipelining](https://s3.bmp.ovh/imgs/2024/01/20/81cc74bf29e1f1b0.png)
    * 即一次性发送多个 payload, 收到对应数量的 ack 后在发送下一组 payload. 如果是阻塞编程, 过多的同时 payload 可能会造成永远的阻塞.
* NAGLE 算法 (NO_DELAY) 以及 TCP 的慢启动
* 消息内容的校验, TCP 的 checksum 比较弱, 可以发送一个随机数的种子, 用于服务端和客户端数据的对比校验

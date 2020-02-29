---
title: "HTTP 协议连接技术演进"
date: 2020-02-29T19:49:54+08:00
draft: true
toc: true
comments: true
tags:
  - untagged
---

> 本文将简述HTTP协议在连接方面的技术演化

参考：

* [博客：HTTP 2.0 协议详解](https://blog.csdn.net/zqjflash/article/details/50179235)
* [博客：HTTP/2笔记之帧](http://www.blogjava.net/yongboy/archive/2015/03/20/423655.html)

## 简述

HTTP 协议是目前使用**最广泛**的应用层协议。

目前主要有3个主流大版本，分别是 `1.0`、`1.1`、`2.0`，且新版本基本兼容之前的版本。

HTTP 协议的核心就是将通讯抽象成无状态的 `Request` 和 `Response` （一次 `请求/响应`）。

HTTP 协议是基于 TCP 之上的协议，可以保证连接是可靠的。保持可靠的连接是有代价的，就是昂贵的三次握手和四次挥手。

在 `HTTP 1.0` 中 每一个 `请求/响应` 都会创建一个 TCP 连接（`短连接`），这样频繁的创建 TCP 连接，有大量的阻塞时间都花在了建立 `TCP 连接` 上。

在 `HTTP 1.1` 中 允许多个 `请求/响应` 共用一个 TCP 连接（`长链接`），这样做减少花费在 建立 TCP 连接的时间。但是 `HTTP 1.1` 的 `长链接` 本质上 是 `电路交换` （每一个 `TCP 连接` 在一段时间内只能 处理一个 `请求/响应`，即 **串行**），无法充分的利用 `TCP 连接`。因此如果一个网站有多个 HTTP 请求，则需要浏览器 创建多个 `TCP 连接` 并行发送请求。

在 `HTTP 2.0` 中 改变了 `请求/响应` 在一个 `HTTP 连接` 上只能串行的缺陷，引入了 `数据帧` 结构，使 `请求/响应` 可以仅在一个 `TCP 连接` 中并行进行。

因此 HTTP 协议在数据传输上的优化的核心连接复用。

## 思想实验

### 约束

* 浏览器的请求并发度为 6
* 假设一个页面加载共有 10 个请求，且所有请求在同一台服务器上
* `TCP 连接` 建立 耗时 20
* 全部带宽下，`Request` 传输时间 1，`Response` 传输时间 2
* 每个请求服务处理时间 10
* 服务器可以并行处理请求
* 忽略 `TCP断连` 时间
* 忽略 `HTTP 2.0` 分帧后头部压缩和帧头部带来的影响
* 假设 `HTTP 2.0` 中每个消息对应一个帧（事实并非如此）
* 下文中数字表示 `TCP连接` 编号，带括号的数字表示 请求编号

### HTTP 1.0

```
TCP连接
1    (1) TCP建连    Request传输  请求处理  Response传输
2    (2) TCP建连    Request传输  请求处理  Response传输
3    (3) TCP建连    Request传输  请求处理  Response传输
4    (4) TCP建连    Request传输  请求处理  Response传输
5    (5) TCP建连    Request传输  请求处理  Response传输
6    (6) TCP建连    Request传输  请求处理  Response传输
7                                                   (7) TCP建连    Request传输  请求处理  Response传输
8                                                   (8) TCP建连    Request传输  请求处理  Response传输
9                                                   (9) TCP建连    Request传输  请求处理  Response传输
10                                                  (10)TCP建连    Request传输  请求处理  Response传输
耗时      20 +      1 * 6 +      10 +    2 * 6 +         20 +      4 * 1 +     10 +     2 * 4 =        90
```

* 因为 HTTP 1.0 是短连接。每个请求都会建立短连接，所以建立连接数 10
* 花费时间 90
    * 在 `(1)~(6)` 号请求中，因为 `传输` 阶段因为是并行的，但是带宽是一定，所以时间需要 `*6`

### HTTP 1.1

通过 `Connection:keep-alive` 请求头，即可开启 `HTTP 1.1` 的长链接

```
TCP连接
1    (1) TCP建连    Request传输  请求处理  Response传输  (7) Request传输  请求处理  Response传输
2    (2) TCP建连    Request传输  请求处理  Response传输  (8) Request传输  请求处理  Response传输
3    (3) TCP建连    Request传输  请求处理  Response传输  (9) Request传输  请求处理  Response传输
4    (4) TCP建连    Request传输  请求处理  Response传输  (10)Request传输  请求处理  Response传输
5    (5) TCP建连    Request传输  请求处理  Response传输
6    (6) TCP建连    Request传输  请求处理  Response传输
     20 +          1 * 6 +      10 +    2 * 6 +           1 * 6 +      10 +    2 * 4 = 72
```

* `HTTP 1.1` 支持长链接 所以建立连接数 6
* 花费时间 72

### HTTP 2.0

```
TCP连接
1(发送通道)      TCP建连  (1)Request传输(2)Request传输(3)Request传输(4)Request传输(5)Request传输(6)Request传输(7)Request传输(8)Request传输(9)Request传输(10)Request传输
1(接收通道)      TCP建连                                                                                                                                          空闲  (1) Response传输 (2) Response传输 (3) Response传输 (4) Response传输 (5) Response传输 (6) Response传输 (7) Response传输 (8) Response传输 (9) Response传输 (10) Response传输
 (服务器处理)                          (1)请求处理
 (服务器处理)                                        (2)请求处理
 (服务器处理)                                                     (3)请求处理
 (服务器处理)                                                                  (4)请求处理
 (服务器处理)                                                                                (5)请求处理
 (服务器处理)                                                                                             (6)请求处理
 (服务器处理)                                                                                                          (7)请求处理
 (服务器处理)                                                                                                                        (8)请求处理
 (服务器处理)                                                                                                                                      (8)请求处理
 (服务器处理)                                                                                                                                                      (10)请求处理
总耗时： 20 + 1 * 10 + 2 * 10 = 50
```

* 因为 `HTTP 2.0` 在传输层 进行分帧，并使用了全新的二进制传输协议，所以建立连接数 1
* 花费时间 50

### 实验结论

通过 上述 的比较，可以看出：

* HTTP 1.1 和 HTTP 1.0 相比
    * `HTTP 1.1` 减少了 `TCP 建连` 所花费的时间
    * 但是并没有提高服务器的吞吐率，因为，不论 `HTTP 1.1` 还是 `HTTP 1.0` 最大连接数 都是 浏览器 配置的 6
    * 数据传输方面，基本一致，都是基于 文本 的串行传输，仅仅是添加了长连接特性
* HTTP 2.0 和 HTTP 1.1 相比
    * 支持了 `多路复用` 特性，可以仅仅在一个 `TCP 连接` 上完成所有的请求，减少了串行请求等待 `服务器处理` 所花费的时间
    * 大大提高了服务端的吞吐率，因为只建立了一个 `TCP 连接`
    * 数据传输方面，是全新的设计，基于 `二进制帧` 的并行传输。因为在传输层的巨大变化，所以 `HTTP 2.0` 才进行了大版本号的更新，而不是使用 `HTTP 1.x` 的命名方式

## HTTP 2.0

在 `HTTP 2.0` 中，应用层的语义和 HTTP 1.1 保持一致。
也就是说 在逻辑上 HTTP 的核心仍然是 无状态的 `Request` 和 `Response`。
应用层可以无需更改逻辑代码即可得到支持。

但在数据传输上引入了新的机制。

在 `HTTP 1.x` 中，`Request` 或 `Response` 的 `Header` 是明文直接 传输的，而 `Body` 可能通过压缩传输。
而且一个 `Request` 或者 `Response` 是整体写入 TCP 流中的，不会进行拆分。
这就决定了 `HTTP 1.x` 无法实现 `多路复用`。

在 `HTTP 2.0` 中，在数据传输方面添加新的 二进制分帧数据层。

### 1、相关概念

* `流`，即一次 `HTTP请求/响应`，一个流会被划分给多个 `帧`
* `消息`，只有两种类型即 `请求` 或者 `响应`，每个 `消息` 可能被划分多个 `帧`
* `帧`，最小通讯单位，包含当前帧属于那个 `流`

### 2、传输过程简述

以下描述忽略部分细节

* 客户端：每个 `请求` 消息 到来后，会为 `请求` 创建一个 `流编号`，并拆分到 成多个 `帧`， 每个帧 中包含这个 `流编号` 和 `帧类型`（主要是 `Header`帧和`Data`帧）。并将帧写到TCP流中
* 服务端：接收到 `帧` 后，会按照 `流编号` 进行分组，同样 `流编号` 的 `帧` 会按照顺序 组装到一起，形成 `请求` 消息
* 服务端：当确定 `请求` 消息的 `Header` 接收完毕后，会调用处理程序，并创建 `响应` 消息
* 服务端：每个 `响应` 消息 到来后，会拆分到 成多个 `帧`， 每个帧 中包含这个 响应对应请求的 `流编号` 和 `帧类型`（主要是 `Header`帧和`Data`帧）。并将帧写到TCP流中
* 客户端：接收到 `帧` 后，会按照 `流编号` 进行分组，同样 `流编号` 的 `帧` 会按照顺序 组装到一起，形成 `响应` 消息
* 客户端：当确定 `响应` 接收完成后，进行后续处理
* 至此 一次 `请求/响应` 处理完成
---
title: reactor
auther: gzj
tags:
  - reactor
categories: 网络编程
description: >-
  网络编程模型通常有如下几种：Reactor, Proactor, Asynchronous, Completion Token, and
  Acceptor-Connector. 本文主要对最主流的Reactor模型进行介绍。通常网络编程模型处理的主要流程如下
abbrlink: 3cf52171
---

## Reactor

### 基于事件的程序设计

事件驱动的好处是占用资源少，效率高，可扩展性强，是支持高性能高并发的不二之选。

如果你熟悉 GUI 编程的话，你就会知道，GUI 设定了一系列的控件，如 Button、Label、文本框等，当我们设计基于控件的程序时，一般都会给 Button 的点击安排一个函数，类似这样：

```c
//按钮点击的事件处理
void onButtonClick(){}
```

这个设计的思想是，一个无限循环的事件分发线程在后台运行，一旦用户在界面上产生了某种操作，例如点击了某个 Button，或者点击了某个文本框，一个事件会被产生并放置到事件队列中，这个事件会有一个类似前面的 onButtonClick 回调函数。事件分发线程的任务，就是为每个发生的事件找到对应的事件回调函数并执行它。这样，一个基于事件驱动的 GUI 程序就可以完美地工作了。

还有一个类似的例子是 Web 编程领域。同样的，Web 程序会在 Web 界面上放置各种界面元素，例如 Label、文本框、按钮等，和 GUI 程序类似，给感兴趣的界面元素设计 JavaScript 回调函数，当用户操作时，对应的 JavaScript 回调函数会被执行，完成某个计算或操作。这样，一个基于事件驱动的 Web 程序就可以在浏览器中完美地工作了。

事件驱动模型，也被叫做反应堆模型（reactor），或者是 Event loop 模型。这个模型的核心有两点。

第一，它存在一个无限循环的事件分发线程，或者叫做 reactor 线程、Event loop 线程。这个事件分发线程的背后，就是 poll、epoll 等 I/O 分发技术的使用。

第二，所有的 I/O 操作都可以抽象成事件，每个事件必须有回调函数来处理。acceptor 上有连接建立成功、已连接套接字上发送缓冲区空出可以写、通信管道 pipe 上有数据可以读，这些都是一个个事件，通过事件分发，这些事件都可以一一被检测，并调用对应的回调函数加以处理。

### 几种 I/O 模型和线程模型设计

任何一个网络程序，所做的事情可以总结成下面几种：

- read：从套接字收取数据；
- decode：对收到的数据进行解析；
- compute：根据解析之后的内容，进行计算和处理；
- encode：将处理之后的结果，按照约定的格式进行编码；
- send：最后，通过套接字把结果发送出去。

### single reactor thread

这里有一张图，解释了这一讲的设计模式。一个 reactor 线程上同时负责分发 acceptor 的事件、已连接套接字的 I/O 事件。

<img src="https://raw.githubusercontent.com/zzugzj/blogImg/master/img/b8627a1a1d32da4b55ac74d4f0230f33.png" alt="img" style="zoom: 80%;" />

### single reactor thread + worker threads

上述的设计模式有一个问题，和 I/O 事件处理相比，应用程序的业务逻辑处理是比较耗时的，这些工作相对而言比较独立，它们会拖慢整个反应堆模式的执行效率。

将这些 decode、compute、encode 型工作放置到另外的线程池中，和反应堆线程解耦。反应堆线程只负责处理 I/O 相关的工作，业务逻辑放到线程池里由空闲的线程来执行。当结果完成后，再交给反应堆线程，由反应堆线程通过套接字将结果发送出去。

<img src="https://raw.githubusercontent.com/zzugzj/blogImg/master/img/7e4505bb75fef4a4bb945e6dc3040823.png" alt="img" style="zoom: 80%;" />

reactor模式虽然可以同时分发Acceptor上的连接建立事件和已建立连接的 I/O 事件，但如果客户端比较多的情况下，单 reactor 线程既分发连接建立，又分发已建立连接的 I/O，有点忙不过来，导致客户端连接成功率偏低。

### 主 - 从 reactor 模式

主 - 从这个模式的核心思想是，主反应堆线程只负责分发 Acceptor 连接建立，已连接套接字上的 I/O 事件交给 sub-reactor 负责分发。其中 sub-reactor 的数量，可以根据 CPU 的核数来灵活设置。而且，同一个套接字事件分发只会出现在一个反应堆线程中，这会大大减少并发处理的锁开销。

<img src="https://raw.githubusercontent.com/zzugzj/blogImg/master/img/9269551b14c51dc9605f43d441c5a92a.png" alt="img" style="zoom: 80%;" />

我们的主反应堆线程一直在感知连接建立的事件，如果有连接成功建立，主反应堆线程通过 accept 方法获取已连接套接字，接下来会按照一定的算法选取一个从反应堆线程，并把已连接套接字加入到选择好的从反应堆线程中。

### 主 - 从 reactor+worker threads 模式

如果说主 - 从 reactor 模式解决了 I/O 分发的高效率问题，那么 work threads 就解决了业务逻辑和 I/O 分发之间的耦合问题。把这两个策略组装在一起，就是实战中普遍采用的模式。

<img src="https://raw.githubusercontent.com/zzugzj/blogImg/master/img/1e647269a5f51497bd5488b2a44444b4.png" alt="img" style="zoom: 80%;" />



这张图解释了主 - 从反应堆下加上 worker 线程池的处理模式。

主 - 从反应堆跟上面介绍的做法是一样的。和上面不一样的是，这里将 decode、compute、encode 等 CPU 密集型的工作从 I/O 线程中拿走，这些工作交给 worker 线程池来处理，而且这些工作拆分成了一个个子任务进行。encode 之后完成的结果再由 sub-reactor 的 I/O 线程发送出去。

### 总结

1：阻塞IO+多进程——实现简单，性能一般

2：阻塞IO+多线程——相比于阻塞IO+多进程，减少了上下文切换所带来的开销，性能有所提高。

3：阻塞IO+线程池——相比于阻塞IO+多线程，减少了线程频繁创建和销毁的开销，性能有了进一步的提高。

4：Reactor+线程池——相比于阻塞IO+线程池，采用了更加先进的事件驱动设计思想，资源占用少、效率高、扩展性强，是支持高性能高并发场景的利器。

5：主从Reactor+线程池——相比于Reactor+线程池，将连接建立事件和已建立连接的各种IO事件分离，主Reactor只负责处理连接事件，从Reactor只负责处理各种IO事件，这样能增加客户端连接的成功率，并且可以充分利用现在多CPU的资源特性进一步的提高IO事件的处理效率。

6：主 - 从Reactor模式的核心思想是，主Reactor线程只负责分发 Acceptor 连接建立，已连接套接字上的 I/O 事件交给 从Reactor 负责分发。其中 sub-reactor 的数量，可以根据 CPU 的核数来灵活设置。
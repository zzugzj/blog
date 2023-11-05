---
title: REST原则六约束与Richardson成熟度模型
auther: gzj
tags:
  - REST
categories: 网络编程
description: >-
  REST即Representational State
  Transfer，表述状态转移。这里Representational被翻译成表述或表征，其含义在网络上存在各种模糊的解释。
abbrlink: d05c43b2
date: 2021-05-21 23:13:49
---

### 前言

REST即Representational State Transfer，表征状态转移。这里Representational被翻译成表述或表征，其含义在网络上存在各种模糊的解释。

Representational State Transfer中，它指的是什么呢？

在计算机领域“Representational”表述或表征的对象是一种资源，这里的资源具体一点可以指图片，视频，数据库中的字段等等，那么这种表述或表征就是定义这些资源的方式，那具体一点，就是JSON，XML这些描述资源的东西。“Representational”是个大的概念，它包括一切的这些JSON，XML的子集。

### REST原则六约束

**（1）Client–server 客户端-服务器模式**

信只能由客户端单方面发起，表现为请求-响应的形式。

**（2）Stateless 无状态**

通信的会话状态（Session State）应该全部由客户端负责维护。

**（3）Cacheable 可缓存**

响应内容可以在通信链的某处被缓存，以改善网络效率。

**（4）Layered system 分层系统**

通过限制组件的行为（即，每个组件只能“看到”与其交互的紧邻层），将架构分解为若干等级的层。

**（5）Code on demand (optional) 按需扩展代码（可选）**

支持通过下载并执行一些代码（例如Java Applet、Flash或JavaScript），对客户端的功能进行扩展。

**（6）Uniform interface 统一接口**

通信链的组件之间通过统一的接口相互通信，以提高交互的可见性。

### Richardson的REST成熟度模型

一个WEB服务有多么的“RESTful”，最有名的就是《RESTful Web Services》的合著者Leonard Richardson提出的REST成熟度模型，简称 Richardson成熟度模型！

**1）资源标识的唯一性（资源的标识） Identification of resources**

每个资源的资源标识可以用来唯一地标明该资源。
请求中的独立资源是可以被标识的，基于web的REST系统使用的URI就是典型的例子。资源在概念上是独立的，并由表述返回客户端。举例来说，服务器把数据库中的数据以HTML，XML 或者 JSON发送给客户端，而这些表述不是在服务中内置的。

**2）资源的自描述性（通过表述对资源执行的操作）Manipulation of resources through these representations**

一个REST系统所返回的资源需要能够描述自身，并提供足够的用于操作该资源的信息，如如何对资源进行添加，删除以及修改等操作。也就是说，一个典型的REST服务不需要额外的文档对如何操作资源进行说明。
当客户端抓取一个资源表述的时候，它都可以通过充足的信息来保证对资源的修改或者删除。

**3）消息的自描述性（自描述消息）Self-descriptive messages**

在REST系统中所传递的消息需要能够提供自身如何被处理的足够信息。例如该消息所使用的MIME类型，是否可以被缓存等。

**4）超媒体驱动性（超媒体作为应用状态的引擎）Hypermedia as the engine of application state**

即客户只可以通过服务端所返回各结果中所包含的信息来得到下一步操作所需要的信息，如：到底是向哪个URL发送请求等。也就是说，一个典型的REST服务不需要额外的文档标示通过哪些URL访问特定类型的资源，而是通过服务端返回的响应来标示到底能在该资源上执行什么样的操作。
一个REST服务的客户端也不需要知道任何有关哪里有什么样的资源这种信息。当客户端抓取一个资源表述的时候，它都可以通过充足的信息来保证对资源的修改或者删除。

**分级解释：**

- **第0级：使用HTTP作为传输方式；一个URI，一个HTTP方法**
  SOAP、XML-RPM都属于这一级别，仅是来回传送"Plain Old XML"(POX)。即使没有显式调用RPC接口（SOAP、XML-RPM），通常会调用服务器端的一个处理过程。一个接口会有一个端点，文档的内容会被解析用还判断所要调用的处理过程及其参数。这种做法相当于把 HTTP 这个应用层协议降级为传输层协议用。HTTP 头和有效载荷是完全隔离的，HTTP 头只用于保证传输，不涉及业务逻辑；有效载荷包含全部业务逻辑，因此 API 可以无视 HTTP 头中的任何信息。

- **第1级：引入了资源的概念，每个资源有对应的标识符和表达；多个URI，一个HTTP方法**
  这些资源仍是被“GETful”接口操作而不是HTTP动词，服务基本上只提供操作这些资源。例如：
  GET [http://example.com/app/createUser]()
  GET [http://example.com/app/getUser?id=123]()
  GET [http://example.com/app/changeUser?id=123&field=value]()
  GET [http://example.com/app/deleteUser?id=123]()

- **第2级：根据语义使用HTTP动词，适当处理HTTP响应状态码；多个URI，多个HTTP方法**
  在这一级别，资源名称为基URI的一部分，而不是查询参数。
  GET（SELECT）：从服务器取出资源（一项或多项）。
  POST（CREATE）：在服务器新建一个资源。
  PUT（UPDATE）：在服务器更新资源（客户端提供改变后的完整资源）。
  PATCH（UPDATE）：在服务器更新资源（客户端提供改变的属性）。
  DELETE（DELETE）：从服务器删除资源。
  HEAD：获取资源的元数据。
  OPTIONS：获取信息，关于资源的哪些属性是客户端可以改变的。

- **第3级：使用超媒体作为应用状态引擎（HATEOAS）；多个URI，多个HTTP方法**
  特别注意，这里是超媒体（hypermedia），超媒体概念是包括超文本的。
  我们已经知道什么是多媒体（multimedia），以及什么是超文本（hypertext）。其中超文本特有的优势是拥有超链接（hyperlink）。如果我们把超链接引入到多媒体当中去，那就得到了超媒体，因此关键角色还是超链接。使用超媒体作为应用引擎状态，意思是应用引擎的状态变更由客户端访问不同的超媒体资源驱动。


Roy Fielding（REST论文的作者）说”只有使用了超媒体的才能算是 REST！，那么第 3 级成熟度以外的都不算 REST！

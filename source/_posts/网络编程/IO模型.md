---
title: IO模型
auther: gzj
tags:
  - IO模型
categories: 网络编程
description: >-
  同步（synchronous） IO和异步（asynchronous） IO，阻塞（blocking）
  IO和非阻塞（non-blocking）IO分别是什么，到底有什么区别？这个问题其实不同的人给出的答案都可能不同，比如wiki，就认为asynchronous
  IO和non-blocking IO是一个东西。
abbrlink: aeafbee0
---

## 阻塞 / 非阻塞

当应用程序调用阻塞IO来完成某个操作时，应用程序会被挂起，比如网络延迟等原因造成的卡顿，这时整个程序感觉就像被阻塞一样。实际上，之所以调用阻塞IO的程序卡顿，是因为内核将CPU的时间切换给其他有需要的进程，如计算，数据复制等操作，应用程序就不会得到CPU，也就进行不了了。

非阻塞 I/O 则不然，当应用程序调用非阻塞 I/O 完成某个操作时，内核立即返回，不会把 CPU 时间切换给其他进程，应用程序在返回后，可以得到足够的 CPU 时间继续完成其他事情。

一张表来总结一下 read 和 write 在阻塞模式和非阻塞模式下的不同行为特性：

![img](https://raw.githubusercontent.com/zzugzj/blogImg/master/img/6e7a467bc6f5985eebbd94ef7de14aaa.png)

关于多路复用和非阻塞IO的区别，多路复用的轮询是内核帮我们完成的，不用像非阻塞IO一样需要我们手动去轮询，另外就是一个进程能同时等待多个文件描述符，而这些文件描述符（套接字描述符）其中的任意一个进入读就绪状态，select()函数就可以返回。

那么，如果在连接数比较低的情况下，使用select/epoll的web server不一定比使用multi-threading + blocking IO的web server性能更好，可能延迟还更大。select/epoll的优势并不是对于单个连接能处理得更快，而是在于能处理更多的连接。

关于accept

当 accept 和 I/O 多路复用 select、poll 等一起配合使用时，如果在监听套接字上触发事件，说明有连接建立完成，此时调用 accept 肯定可以返回已连接套接字。这样看来，似乎把监听套接字设置为非阻塞，没有任何好处。

如果在监听套接字上有可读事件发生时，并没有马上调用 accept。由于客户端发生了 RST 分节，该连接被接收端内核从自己的已完成队列中删除了，此时再调用 accept，由于没有已完成连接，accept 一直阻塞，更为严重的是，该线程再也没有机会对其他 I/O 事件进行分发，相当于该服务器无法对其他 I/O 进行服务。如果我们将监听套接字设为非阻塞，上述的情形就不会再发生。只不过对于 accept 的返回值，需要正确地处理各种看似异常的错误，例如忽略 EWOULDBLOCK、EAGAIN 等。

### 阻塞IO和进程模型

#### 父进程和子进程

一个进程有完整的地址空间、程序计数器等，如果想创建一个新的进程，使用函数 fork 就可以。

```c
pid_t fork(void)
返回：在子进程中为0，在父进程中为子进程ID，若出错则为-1
```

程序调用 fork 一次，它却在父、子进程里各返回一次（fork前是一个进程执行，fork后就是两个进程执行了，所以返回两次）。在调用该函数的进程（即为父进程）中返回的是新派生的进程 ID 号，在子进程中返回的值为 0。想要知道当前执行的进程到底是父进程，还是子进程，只能通过返回值来进行判断。

fork 函数实现的时候，实际上会把当前父进程的所有相关值都克隆一份，包括地址空间、打开的文件描述符、程序计数器等，就连执行代码也会拷贝一份，新派生的进程的表现行为和父进程近乎一样。

```c
if(fork() == 0){
  do_child_process(); //子进程执行代码
}else{
  do_parent_process();  //父进程执行代码
}
```

当一个子进程退出时，系统内核还保留了该进程的若干信息，比如退出状态。这样的进程如果不回收，就会变成僵尸进程。在 Linux 下，这样的“僵尸”进程会被挂到进程号为 1 的 init 进程上。所以，由父进程派生出来的子进程，也必须由父进程负责回收，否则子进程就会变成僵尸进程。僵尸进程会占用不必要的内存空间，如果量多到了一定数量级，就会耗尽我们的系统资源。

使用阻塞 I/O 和进程模型，为每一个连接创建一个独立的子进程来进行服务，是一个非常简单有效的实现方式，这种方式可能很难满足高性能程序的需求，但好处在于实现简单。在实现这样的程序时，需要注意两点：要注意对套接字的关闭梳理；要注意对子进程进行回收，避免产生不必要的僵尸进程。

## IO多路复用之select、poll、epoll

I/O 多路复用的设计初衷就是解决这样的场景。我们可以把标准输入、套接字等都看做 I/O 的一路，多路复用的意思，就是在任何一路 I/O 有“事件”发生的情况下，==通知==应用程序去处理相应的 I/O 事件，这样我们的程序就可以在同一时刻仿佛可以处理多个 I/O 事件。

select，poll，epoll本质上都是同步I/O，因为他们都需要在读写事件就绪后自己负责进行读写，也就是说这个读写过程是阻塞的，而异步I/O则无需自己负责进行读写，异步I/O的实现会负责把数据从内核拷贝到用户空间。

### select

select 函数是一种常见的 I/O 多路复用技术。使用 select 函数，通知内核挂起进程，当一个或多个 I/O 事件发生后，控制权返还给应用程序，由应用程序进行 I/O 事件的处理。

这些 I/O 事件的类型非常多，比如：

- 标准输入文件描述符准备好可以读。
- 监听套接字准备好，新的连接已经建立成功。
- 已连接套接字准备好可以写。
- 如果一个 I/O 事件等待超过了 10 秒，发生了超时事件。

#### select 函数的使用方法

select 方法是多个 UNIX 平台支持的非常常见的 I/O 多路复用技术，其良好跨平台支持也是它的一个优点。它通过描述符集合来表示检测的 I/O 对象，通过三个不同的描述符集合来描述 I/O 事件 ：可读、可写和异常。但是 select 有一个缺点，那就是所支持的文件描述符的个数是有限的。在 Linux 系统中，select 的默认最大值为 1024。

函数声明：

```c
int select(int maxfd, fd_set *readset, fd_set *writeset, fd_set *exceptset, const struct timeval *timeout);
返回：若有就绪描述符则为其数目，若超时则为0，若出错则为-1
```

在这个函数中，maxfd 表示的是待测试的描述符基数，它的值是待测试的最大描述符加 1。

紧接着的是三个描述符集合，分别是读描述符集合 readset、写描述符集合 writeset 和异常描述符集合 exceptset，这三个分别通知内核，在哪些描述符上检测数据可以读，可以写和有异常发生。

这些描述符集合的控制可以使用下面的宏：

```c
void FD_ZERO(fd_set *fdset);　　　　　　
void FD_SET(int fd, fd_set *fdset);　　
void FD_CLR(int fd, fd_set *fdset);　　　
int  FD_ISSET(int fd, fd_set *fdset);
```

关于这些宏可以这样理解，把描述符集合想象为一个向量（数组），这个向量的每个元素都是二进制数中的 0 或者 1：

```c
a[maxfd-1], ..., a[1], a[0]
```

那么：

- FD_ZERO 用来将这个向量的所有元素都设置成 0；

- FD_SET 用来把对应套接字 fd 的元素，a[fd]设置成 1；

- FD_CLR 用来把对应套接字 fd 的元素，a[fd]设置成 0；

- FD_ISSET 对这个向量进行检测，判断出对应套接字的元素 a[fd]是 0 还是 1。

其中 0 代表不需要处理，1 代表需要处理。

### poll

#### poll 函数介绍

poll 是除了 select 之外，另一种普遍使用的 I/O 多路复用技术，和 select 相比，它和内核交互的数据结构有所变化，另外，也突破了文件描述符的个数限制。poll函数原型：

```c
int poll(struct pollfd *fds, unsigned long nfds, int timeout); 
　　　
返回值：若有就绪描述符则为其数目，若超时则为0，若出错则为-1
```

这个函数里面输入了三个参数，第一个参数是一个 pollfd 的数组。其中 pollfd 的结构如下：

```c
struct pollfd {
    int    fd;       /* file descriptor */
    short  events;   /* events to look for */
    short  revents;  /* events returned */
 };
```

pollfd结构包含了要监视的event和发生的event，不再使用select“参数-值”传递的方式。同时，pollfd并没有最大数量限制（但是数量过大后性能也是会下降）。 和select函数一样，poll返回后，需要轮询pollfd来获取就绪的描述符。

> 从上面看，select和poll都需要在返回后，`通过遍历文件描述符来获取已经就绪的socket`。事实上，同时连接的大量客户端在一时刻可能只有很少的处于就绪状态，因此随着监视的描述符数量的增长，其效率也会线性下降。##

### epoll

这张图可以更直观的观察这三种多路复用的性能优劣：

![img](https://raw.githubusercontent.com/zzugzj/blogImg/master/img/fd2e25f72a5103ef78c05c7ad2dab060.png)

#### epoll的用法

epoll 可以说是和 poll 非常相似的一种 I/O 多路复用技术。epoll 通过监控注册的多个描述字，来进行 I/O 事件的分发处理。不同于 poll 的是，epoll 不仅提供了默认的 level-triggered（条件触发）机制，还提供了性能更为强劲的 edge-triggered（边缘触发）机制。

使用 epoll 进行网络程序的编写，需要三个步骤，分别是 epoll_create，epoll_ctl 和 epoll_wait。

##### epoll_create

```c
int epoll_create(int size);
```

epoll_create() 方法创建了一个 epoll 实例。这个 epoll 实例被用来调用 epoll_ctl 和 epoll_wait，如果这个 epoll 实例不再需要，比如服务器正常关机，需要调用 close() 方法释放 epoll 实例，这样系统内核可以回收 epoll 实例所分配使用的内核资源。

关于参数 size，在一开始的 epoll_create 实现中，是用来告知内核期望监控的文件描述字的数量，然后内核使用这部分的信息来初始化内核数据结构，在新的实现中，这个参数不再被需要，因为内核可以动态分配需要的内核数据结构。我们只需要注意，每次将 size 设置成一个大于 0 的整数就可以了。

##### epoll_ctl

```c
 int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event);
        返回值: 若成功返回0；若返回-1表示出错
```

调用 epoll_ctl 可以往这个 epoll 实例增加或删除监控的事件。

第一个参数 epfd 是刚刚调用 epoll_create 创建的 epoll 实例描述字，可以简单理解成是 epoll 句柄。

第二个参数表示增加还是删除一个监控事件，它有三个选项可供选择：添加EPOLL_CTL_ADD，删除EPOLL_CTL_DEL，修改EPOLL_CTL_MOD。分别添加、删除和修改对fd的监听事件。

第三个参数是注册的事件的文件描述符，比如一个监听套接字。

第四个参数表示的是注册的事件类型，并且可以在这个结构体里设置用户需要的数据，其中最为常见的是使用联合结构里的 fd 字段，表示事件所对应的文件描述符。

```c
typedef union epoll_data {
     void        *ptr;
     int          fd;
     uint32_t     u32;
     uint64_t     u64;
 } epoll_data_t;

 struct epoll_event {
     uint32_t     events;      /* Epoll events */
     epoll_data_t data;        /* User data variable */
 };
//events可以是以下几个宏的集合：
EPOLLIN：表示对应的文件描述字可以读；
EPOLLOUT：表示对应的文件描述字可以写；
EPOLLRDHUP：表示套接字的一端已经关闭，或者半关闭；
EPOLLHUP：表示对应的文件描述字被挂起；
EPOLLET：设置为 edge-triggered，默认为 level-triggered。(边缘触发、水平触发)
```

##### epoll_wait

```c
int epoll_wait(int epfd, struct epoll_event *events, int maxevents, int timeout);
  返回值: 成功返回的是一个大于0的数，表示事件的个数；返回0表示的是超时时间到；若出错返回-1.
```

epoll_wait() 函数类似之前的 poll 和 select 函数，调用者进程被挂起，在等待内核 I/O 事件的分发。

这个函数的第一个参数是 epoll 实例描述字，也就是 epoll 句柄。

第二个参数返回给用户空间需要处理的 I/O 事件，这是一个数组，数组定义的大小就是maxevents，==数组含义元素的数目由 epoll_wait 的返回值决定==，这个数组的每个元素都是一个需要待处理的 I/O 事件，其中 events 表示具体的事件类型，事件类型取值和 epoll_ctl 可设置的值一样，这个 epoll_event 结构体里的 data 值就是在 epoll_ctl 那里设置的 data，也就是用户空间和内核空间调用时需要的数据。

第三个参数是一个大于 0 的整数，表示 epoll_wait 可以返回的最大事件值。

第四个参数是 epoll_wait 阻塞调用的超时值，如果这个值设置为 -1，表示不超时；如果设置为 0 则立即返回，即使没有任何 I/O 事件发生。

##### edge-triggered VS level-triggered

　　**LT模式**：当epoll_wait检测到描述符事件发生并将此事件通知应用程序，`应用程序可以不立即处理该事件`。下次调用epoll_wait时，会再次响应应用程序并通知此事件。
　　**ET模式**：当epoll_wait检测到描述符事件发生并将此事件通知应用程序，`应用程序必须立即处理该事件`。如果不处理，下次调用epoll_wait时，不会再次响应应用程序并通知此事件。

ET模式在很大程度上减少了epoll事件被重复触发的次数，因此效率要比LT模式高。epoll工作在ET模式的时候，必须使用非阻塞套接口，以避免由于一个文件句柄的阻塞读/阻塞写操作把处理多个文件描述符的任务饿死。

##### epoll 的性能分析

**边缘触发和水平触发：**

如果某个套接字有 100 个字节可以读，边缘触发（edge-triggered）和条件触发（level-triggered）都会产生 read ready notification 事件，如果应用程序只读取了 50 个字节，边缘触发就会陷入等待；而条件触发则会因为还有 50 个字节没有读取完，不断地产生 read ready notification 事件。

在条件触发下（level-triggered），如果某个套接字缓冲区可以写，会无限次返回 write ready notification 事件，在这种情况下，如果应用程序没有准备好，不需要发送数据，一定需要解除套接字上的 ready notification 事件，否则 CPU 就直接跪了。

我们简单地总结一下，边缘触发只会产生一次活动事件，性能和效率更高。不过，程序处理起来要更为小心。

**epoll和poll/select的对比：**

poll/select 先将要监听的 fd 从用户空间拷贝到内核空间, 然后在内核空间里面进行处理之后，再拷贝给用户空间。这里就涉及到内核空间申请内存，释放内存等等过程，这在大量 fd 情况下，是非常耗时的。而 epoll 维护了一个红黑树，通过对这棵黑红树进行操作，可以避免大量的内存申请和释放的操作，而且查找速度非常快。

第二，select/poll 从休眠中被唤醒时，如果监听多个 fd，只要其中有一个 fd 有事件发生，内核就会遍历内部的 list 去检查到底是哪一个事件到达，并没有像 epoll 一样, 通过 fd 直接关联 eventpoll 对象，快速地把 fd 直接加入到 eventpoll 的就绪列表中。

epoll 维护了一棵红黑树来跟踪所有待检测的文件描述字，黑红树的使用减少了内核和用户空间大量的数据拷贝和内存分配，大大提高了性能。

同时，epoll 维护了一个链表来记录就绪事件，内核在每个文件有事件发生时将自己登记到这个就绪事件列表中，通过内核自身的文件 file-eventpoll 之间的回调和唤醒机制，减少了对内核描述字的遍历，大大加速了事件通知和检测的效率，这也为 level-triggered 和 edge-triggered 的实现带来了便利。
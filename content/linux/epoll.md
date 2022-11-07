---
title: "深入理解Linux中网络I/O复用并发模型"
date: 2022-11-05T17:17:17+17:17
draft: false
---

<!--more-->

## 本文说明

本文所有内容来自于 [AceId](https://github.com/aceld) 如有侵权，联系作者删除！

## 基本概念

### 流

- 可以进行I/O操作的内核对象
- 文件、管道、套接字......
- 流的入口：文件描述符(fd)

### I/O操作

所有对流的读写操作，都可以称之为I/O操作

### 阻塞

- 阻塞等待：不占用CPU宝贵的时间片
  - 缺点：不能够很好的处理多个(I/O)请求的问题
  - 同一个阻塞，同一时刻只能处理一个流的阻塞监听
- 非阻塞忙轮询 ：占用CPU，系统资源

> 在处理意见数据的接收场景时，建议优先使用阻塞等待的方式，不浪费性能资源

### 多路IO复用

- 既能够阻塞等待，不浪费资源
- 也能够同一时刻监听多个IO请求的状态

## 解决阻塞死等待的缺点

1. 阻塞等待+多进程/多线程：需要开辟线程浪费资源
2. 非阻塞+忙轮询：
   1. CPU在大量的做判读，while 和 for
   2. CPU的利用率不高

```c
while true {
	for i in 流[] {
		if i has 数据 {
			读 或者 其他处理
		}
	}
}
```

3. 多路IO复用机制 -- select
   1. 监听的IO数量有限，默认是1024个
   2. 不会精准的告诉开发者，哪些IO是可读可写的，需要遍历

```c
while true {
	select(流[]); // 阻塞
	// 有消息抵达
	for i in 流[] {
		if i has 数据 {
			读 或者 其他处理
		}
	}
}
```

3. 多路IO复用机制 -- epoll
   1. 与 select，poll 一样，对I/O多路复用的技术
   2. 只关心“活跃”的链接，无需遍历全部描述集合
   3. 能够处理大量的链接请求(系统可以打开的文件数目， 可以通过/proc/sys/fd/file-max查看)

```c
while true {
	可处理的流[] = epoll_wait(epoll_fd) // 阻塞
	
	// 有消息抵达,全部放在 “可处理的流[]”中
	for i in 可处理的流[] {
		读 或者 其他处理
	}
}
```

## Epoll 的API

### 创建EPOLL

```c
// epoll_create 创建一个EPOLL实例。返回新实例的FD。“Size”参数是一个提示，指定要与新实例关联的文件描述符的数量。
// EPOLL_CREATE()返回的FD应该用Close()关闭。
// size 内核监听的数目
// return 返回一个epoll句柄(即一个文件描述符)
int epoll_create (int __size);
```

### 控制EPOLL

```c
/* Valid opcodes ( "op" parameter ) to issue to epoll_ctl().  */
#define EPOLL_CTL_ADD 1	/* Add a file descriptor to the interface.  */
#define EPOLL_CTL_DEL 2	/* Remove a file descriptor from the interface.  */
#define EPOLL_CTL_MOD 3	/* Change file descriptor epoll_event structure.  */
/* 
	 Manipulate an epoll instance "epfd". Returns 0 in case of success,
   -1 in case of error ( the "errno" variable will contain the
   specific error code ) The "op" parameter is one of the EPOLL_CTL_*
   constants defined above. The "fd" parameter is the target of the
   operation. The "event" parameter describes which events the caller
   is interested in and any associated user data.
*/
extern int epoll_ctl (int __epfd, int __op, int __fd, struct epoll_event *__event)

enum EPOLL_EVENTS
  {
    EPOLLIN = 0x001,
#define EPOLLIN EPOLLIN
    EPOLLPRI = 0x002,
#define EPOLLPRI EPOLLPRI
    EPOLLOUT = 0x004,
#define EPOLLOUT EPOLLOUT
    EPOLLRDNORM = 0x040,
#define EPOLLRDNORM EPOLLRDNORM
    EPOLLRDBAND = 0x080,
#define EPOLLRDBAND EPOLLRDBAND
    EPOLLWRNORM = 0x100,
#define EPOLLWRNORM EPOLLWRNORM
    EPOLLWRBAND = 0x200,
#define EPOLLWRBAND EPOLLWRBAND
    EPOLLMSG = 0x400,
#define EPOLLMSG EPOLLMSG
    EPOLLERR = 0x008,
#define EPOLLERR EPOLLERR
    EPOLLHUP = 0x010,
#define EPOLLHUP EPOLLHUP
    EPOLLRDHUP = 0x2000,
#define EPOLLRDHUP EPOLLRDHUP
    EPOLLEXCLUSIVE = 1u << 28,
#define EPOLLEXCLUSIVE EPOLLEXCLUSIVE
    EPOLLWAKEUP = 1u << 29,
#define EPOLLWAKEUP EPOLLWAKEUP
    EPOLLONESHOT = 1u << 30,
#define EPOLLONESHOT EPOLLONESHOT
    EPOLLET = 1u << 31
#define EPOLLET EPOLLET
  };

struct epoll_event
{
  uint32_t events;	/* Epoll events */
  epoll_data_t data;	/* User data variable */
} __EPOLL_PACKED;
  
typedef union epoll_data
{
  void *ptr;
  int fd;
  uint32_t u32;
  uint64_t u64;
} epoll_data_t;
```

### 等待EPOLL

```c
/* Same as epoll_wait, but the thread's signal mask is temporarily
   and atomically replaced with the one provided as parameter.
   This function is a cancellation point and therefore not marked with
   __THROW.  */
extern int epoll_pwait (int __epfd, struct epoll_event *__events, int __maxevents, int __timeout, const __sigset_t *__ss);
```

### 使⽤用epoll编程主流程⻣骨架

```c
// 创建epoll
int epfd = epoll_create(1000);

// 将listen_fd 添加进 epoll 中
epoll_ctl(epfd, EPOLL_CTL_ADD, listen_fd, &listen_event);


while (1) {
  // 阻塞等待epoll中的fd触发
  int artive_cnt = epoll_wait(epfd, events, 1000, -1);
  for (i = 0; i < active_cnt; i++) {
    if (event[i].data.fd == listen_fd) {
      // accept. 并且将accept的fd加进epoll中。
    } else if (event[i].events & EPOLLIN) {
      // 对此fd 进行读操作
    } else if (event[i].events & EPOLLOUT) {
      // 对此fd 进行写操作
    }
  }
}
```

## 触发模式

> 默认是水平触发模式，如果要边缘触发模式，要在给事件进行绑定的时候通过 `EPOOLET` 来设置

### 水平触发

如果用户在监听 `epoll` 事件，当内核有事件的时候，会拷贝给用户态事件，但是如果用户只处理了一次，那么剩下没有处理的会在下一次 `epoll_wait` 再次返回该事件。

### 边缘触发

相比跟水平触发相反，当内核有事件达到，只会通知用户一次，至于用户处理还是不处理，以后将不会再通知，这样减少了拷贝过程，增加了性能，但是相对来说，如果用户马虎忘记处理，将会产生事件丢的情况

## 单线程Accept(⽆O复用)

![epoll_1](/linux/epoll_1.png)

### 模型分析

1. 主线程 main thread 执行阻塞Accept， 每次客户端Connect链接过来， main thread中accept响应并建立连接。
2. 创建链接成功， 得到Connfd1套接字后，依然在main thread串行处理套接字读写，并处理业务。
3. 在2处理业务中，如果有新客户端Connect过来，Server无响应，直到当前套接字全部业务处理完毕。
4. 当前客户端处理完后，完毕链接，处理下一个客户端请求。

### 优点

socket编程流程清晰简单那，适合学习使用，了解socket基本编程流程。

### 缺点

1. 该模型并非并发模型，是串行的服务器，同一时刻，监听并响应最大的网络请求量为1。即并发量为1。

2. 仅适合学习基本socket编程，不适合任何服务器Server构建。

## 单线程Accept+多线程读写业务(⽆IO复用)

![epoll_2](/linux/epoll_2.png)

### 模型分析

1. 主线程main thread 执行阻塞Accept， 每次客户端Connect链接过来，main thread 中accept响应并建立连接
2. 创建链接成功，得到Connfd1套接字后，创建一个新线程thread1用来处理客户端的读写业务。main thread依旧回到Accept阻塞等待新客户端。
3. thread1通过套接字Connfd1与客户端进行通信读写。
4. server在2处理业务中，如果有新客户端Connect过来，main thread中Accept依然响应并建立连接， 重复2过程

### 优点

1. 基于 [单线程Accept(⽆O复用)](#单线程accepto复用) , 支持了并发的特性
2. 使用比较灵活，一个客户端对应一个线程单独处理，server处理业务的内聚性比较高，客户端无论如何写、服务端均会有一个线程做资源响应

### 缺点

1. 随着客户端的数量增多，需要开辟的线程的数量也增加，客户端和server线程的数量1:1正比关系。因此对于高并发场景，现成数量受到硬件的瓶颈。线程过多也会增加CPU的切换成本，降低CPU的利用率。
2. 对于长链接，客户端一旦无业务读写，只要不关闭，server就应该对保持这个连接的状态(心跳检测，健康检查机制)，占用连接资源和线程的开销
3. 仅适合客户端数量不大，并且是可控的场景来使用
4. 适合学习基本的socket编程，不适合做并发服务器

## 单线程多路IO复⽤

![epoll_3](/linux/epoll_3.png)

### 模型分析

1. 主线程main thread创建listenFd之后，采用多路I/O复用机制(select、epoll)进行IO状态阻塞监控。有Client1客户端Connect请求，I/O复用机制检车ListenFd触发读时间，则进行Accept建立连接，并将新生成的connFd1加入到监听I/O集合中。
2. Client1再次进行正常读写业务请求，main thread的多路I/O复用机制阻塞返回，会触该套接字的读/写事件等。
3. 对于Client1的读写业务，Server依旧在main thread执行流程中继续执行，此时如果有新的客户端COnnect链接请求过来，Server将没有即时响应。
4. 等到Server处理完一个链接的Read+Write操作，继续回到多路I/O复用机制阻塞，其他链接过来重复2、3流程

### 优点

1. 单线程解决了可以监听多个客户端读写状态，不需要1:1客户端的线程数量关系
2. 多路I/O复用，阻塞，非忙轮询状态，不浪费CPU资源，对CPU利用率较高

### 缺点

1. 虽然可以监听多个客户端读写状态，但是同一时间，只能够处理一个客户端的读写操作，实际上读写业务并发为1

2. 当多个客户端访问server， 业务是串行执行，大量请求的会有排队延迟的现象。比如Client3占据main thread流程时，Client1和Client2流程会卡在IO复用等待下次监听触发事件。


## 单线程多路IO复用+多线程读写业务(业务工作池)

![epoll_4](/linux/epoll_4.png)

### 模型分析

1. 主线程main thread创建listenFd之后，采用多路I/O复用机制(select、epoll)进行IO状态阻塞监控，有Client客户端Connect请求，I/O复用机制检测到ListenFd触发时间，则进行Accept建立连接，并将新生成的connFd加入到监听I/O集合中。
2. 当connFd1有可读消息，触发读事件，并且进行读写消息
3. main thread按照固定的协议读取消息，并且交给worker poll工作线程池，工作线程池在server启动之前就已经开启固定数量的thread，里面的写成只处理消息业务，不进行套接字读写操作。
4. 工作池处理完业务，触发connFd1写事件，将回执客户端的消息通过main thread写给对方。

### 优点

1. 对于 [单线程多路io复⽤](#单线程多路io复) 将业务的处理部分，通过工作池分离出来。能够减少客户端访问Server导致业务串行会有大量请求排队的延迟时间
2. 实际上读写的业务并发为1，但是业务流程的并发为word pool线程数量，加快了业务处理的并行效率。

### 缺点

1. 读写依旧是main thread单独处理，最高的读写并行通道依然为1
2. 虽然多个worker线程处理业务，但是最后返回给客户端依旧也需要排队。因为出口还是main thread的read+write 1个通道

## 单线程IO复用+多线程IO复⽤(链接线程池)

![epoll_5](/linux/epoll_5.png)

### 模型分析

1. Server在启动监听之间，开辟固定数量(N)的线程，用Thread Pool线程池管理
2. 主线程main thread 创建listenFd之后，采用多路I/O复用机制(select，epoll)进行IO状态阻塞监控。有Client1客户端Connect请求，I/O复用机制检测到ListenFd触发读事件，则进行Accept建立连接，并将新生成的connFd1分发给Thread Pool中的某个线程进行监听。
3. Thread Pool中的每个thread都启动多路I/O复用机制(select、epoll)，用来监听main thread 建立成功并且分发下来的socket套接字
4. 如图，thread1监听ConnFd1、ConnFd2，thread2监听ConnFd3，thread3监听ConnFd4， 当对应的ConnFd有读写事件，对应的线程处理该套接字的读写及业务。

### 优点

1. 将之前  [单线程多路io复⽤](#单线程多路io复) ，分散到多线程类完成，这样就增加了同一时刻读写的并行通道，并行通道的数量N，N是线程池线程的数量

2. Server同事监听ConnFd套接字的数量，几乎是成倍增加，之前的全部的监控数量取决于main thread的多路IO复用机制的最大限制(select 1024, epoll默认与内存大小有关，约3 ~ 6w不等)，所以理论单点Server最高响应并发数量N*(3 ~ 6w) 

   > N是线程池的线程数量，建议线程的数量和CPU核心的数量比例是1:1

3. 如果良好的线程池数量和CPU核心数适配，那么可以尝试CPU核心与Thread进行绑定，从而减低CPU的切换频率，提升每个Thread处理合理业务的效率，降低CPU的切换成本

### 缺点

1. 虽然监听的并发数量提升，但是最高的读写并行通道依然为N，并且多个身处与同一个Thread的客户端，会出现读写延迟现象。实际上每个Thread模型的特征与  [单线程多路io复⽤](#单线程多路io复) 单线程多路IO复用机制是一致的。

## 单进程多路I/O复⽤+多进程多路I/O复用(进程池)

![epoll_6](/linux/epoll_6.png)

### 模型分析

1. 与 [单线程IO复用+多线程IO复⽤(链接线程池)](#单线程io复用多线程io复链接线程池) 无大差异
2. 不同点：
   1. 进程和线程的内存不同导致，main process(主进程) 不再Accept操作，而是将Accept过程分散给各个子进程(process)中。
   2. 进程的特性，资源独立，所以main process如果Accept成功的fd，其他进程无法共享资源，所以要各子进程自行Accept创建链接
   3. main process只是监听ListenFd状态，一旦触发读事件(有新连接请求)，通过一些IPC(进程间通信：如信号、共享内存、管道)等，让各自子进程Process竞争Accept完成链接建立，并各自监听

### 优缺点

1. 与 [单线程IO复用+多线程IO复⽤(链接线程池)](#单线程io复用多线程io复链接线程池) 无大差异

2. 不同点：
   1. 多进程内存资源占用稍微大一些
   2. 多进程模型安全稳定型比较强，这也是因为各自进程互不干扰的特点导致

## 单线程多路I/O复用+多线程多路I/O复用+多线程

![epoll_7](/linux/epoll_7.png)

### 模型分析

1. Server在启动监听之前，开辟固定数量(N)的线程，用Thread Pool线程池管理
2. 主线程main thread创建ListenFd之后，采用多路I/O复用机制(如：select、epoll)进行IO状态阻塞监控。有Client1客户端Connect请求，I/O复用机制检测到ListenFd触发读事件，则进行Accept建立连接，并将新生成的ConnFd1分发给Thread Pool中的某个线程进行监听。
3. Thread Pool中的每个Thread都启动多路I/O复用机制(select、epoll),用来监听main thread建立成功并分发下来的socket套接字。一旦其中某个被监听的客户端套接字触发I/O读写业务
4. 当某个读写线程完成当前读写业务，如果当前套接字没有被关闭，那么将当前客户端套接字如ConnFd3重新加回线程自我销毁

### 优点

1. 在 [单线程IO复用+多线程IO复⽤(链接线程池)](#单线程多路io复用多线程读写业务业务工作池) 基础上，除了能够保证同事相应最高的并发数，又能够解决读写并行通道的局限问题
2. 同一时刻的读写并行通道，达到了最大化极限，一个客户端可以对应一个单独的执行流程处理读写业务，读写并行通道与客户端的数量1:1关系

### 缺点

1. 该模型过于理想化，因为要求CPU核心数数量足够大。
2. 如果硬件CPU数量可数，那么该模型就造成大量CPU切换的成本浪费。因为为了保证读写并行通道和客户端是1:1的关系，就要保证server开辟的thread的数量与客户端一致。

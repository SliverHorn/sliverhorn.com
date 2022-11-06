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

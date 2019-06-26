# I/O多路复用技术

* [1. select系统调用](#1)
* [2. poll系统调用](#2)
* [3. epoll系统调用](#3)

* I/O复用使得程序能同时监听多个文件描述符，这对提高程序的性能至关重要。通常，网络程序在下列情况下需要使用I/O复用技术:
    1. 客户端程序要同时处理多个socket。
    2. 客户端程序要同时处理用户输入和网络连接。
    3. TCP服务器要同时处理监听socket和连接socket。这是I/O复用使用最多的场合。
    4. 服务器要同时处理TCP请求和UDP请求。
    5. 服务器要同时监听多个端口，或者处理多种服务。

<h2 id="1">1. select系统调用</h2>

* select系统调用的原型

```
#include<sys/select.h>
int select(int nfds, fd_set* readfds, fd_set* writefds, fd_set* exceptfds, struct timeval* timeout);
```

nfds: 指定被监听的文件描述符的总数。它通常被设置为select监听的所有文件描述符中的最大值加1，因为文件描述符是从0开始计数的。

readfds, writefds和exceptfds参数分别指向可读，可写和异常等事件对应的文件描述符集合。应用程序调用select函数时，通过这3个参数传入自己感兴趣的文件描述符。select调用返回时，内核将修改他们来通知应用程序那些文件描述符已经就绪，这三个参数是fd_set结构体指针类型。

```
#include <typesizes.h>

#define __FD_SETSIZE 1024
#include <sys/select.h>
#define FD_SETSIZE __FD_SETSIZE
typedef long int __fd_mask;
#undef __NFDBITS
#define __NFDBITS ( 8 * (int) sizeof (__fd_mask) )
typedef struct
{
    #ifdef __USE_XOPEN
        __fd_mask fds_bits[ __FD_SETSIZE / __NFDBITS ];
    #define __FDS_BITS(set) ((set)->fds_bits)
    #else
        __fd_mask __fds_bits[ __FD_SETSIZE / __NFDBITS ];
    #define __FDS_BITS(set) ((set)->__fds_bits)
    #endif
} fd_set;
```

由于位操作过于烦琐，我们应该使用下面的一系列宏来访问fd_set结构体中的位：

```
#include <sys/select.h>
FD_ZERO(fd_set *fdset); /*清除fdset的所有位*/
FD_SET(int fd, fd_set *fdset);  /*设置fdset的位fd*/
FD_CLR(int fd, fd_set *fdset);  /*清除fdset的位fd*/
int FD_ISSET(int fd, fd_set *fdset);    /*测试fdset的位fd是否被设置*/
```

timeout：用来设置select函数的超时时间。它是一个timeval结构类型的指针，采用指针参数是因为内核将修改它以告诉应用程序select等待了多久。不过我们不能完全信任select调用返回后的timeout值，比如调用失败时timeout值是不确定的。

```
struct timeval
{
    long tv_sec;    /*秒数*/
    long tv_usec;   /*微妙数*/
};
```

如果timeout变量的tv_sec和tv_usec成员都传递0，则select将立即返回。如果给timeout传递NULL，则select将一直阻塞，直到某个文件描述符就绪。
select成功时返回就绪(可读，可写和异常)文件描述符的总数。如果在超时时间内没有任何文件描述符就绪，select将返回0.select失败时返回-1并设置errno。如果在select等待期间，程序接收到信号，则select立即返回-1，并设置errno为EINTR。

* 文件描述符就绪条件

    1. socket内核接收缓存区中的字节数大于或等于其低水位标记SO_RCVLOWAT.此时我们可以无阻塞地读该socket，并且读操作返回的字节数大于0.
    2. socket通信的对方关闭连接。此时对该socket的读操作将返回0.
    4. 监听socket上有新的连接请求。
    5. socket上有未处理的错误。此时我们可以使用getsockopt来读取和清除该错误。
    6. socket内核发送缓存区中的可用字节数大于或等于其低水位标记SO_SNDLOWAT.此时我们可以无阻塞的写该socket，并且写操作返回的字节数大于0.
    7. socket的写操作被关闭。对写操作被关闭的socket执行写操作将触发一个SIGPIPE信号。
    8. socket使用非阻塞connect连接成功或者失败(超时)之后。
    9. socket上有未处理的错误。此时我们可以使用getsockopt来读取和清除该错误。


<h2 id="2">2. poll系统调用</h2>

* poll系统调用

poll系统调用和select类似，也是在指定时间内轮询一定数量的文件描述符，以测试其中是否有就绪者。

```
#include <poll.h>
int poll(struct pollfd* fds, nfds_t nfds, int timeout);
```

fds: 是一个pollfd结构类型的数组，它指定所有我们感兴趣的文件描述符上发生的可读，可写和异常等事件。

```
struct pollfd
{
    int fd; /*文件描述符*/
    short events;   /*注册的事件 - 事件可或*/
    short revents;  /*实际发生的事件，由内核填充*/
};
```

![events](./img/poll0.png)
![events](./img/poll1.png)

要使用POLLRDHUP事件时，需要在代码最开始处定义_GNU_SOURCE.

nfds: 指定被监听事件集合fds的大小。(typedef unsigned long int nfds_t)

timeout: 指定poll的超时值，单位是毫秒。当timeout为-1时，poll调用将永远阻塞，直到某个事件发生：当timeout为0时，poll调用将立即返回。


* epoll系统调用

```
#include<sys/epoll.h>
int epoll_create(int size)
```

size: 只是给内核一个提示，告诉它事件表需要多大。该函数返回的文件描述符将用作其他所有epoll系统调用的第一个参数。

```
int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event)
```

fd: 是要操作的文件描述符。

op: 指定操作类型。

![op](./img/op.png)

event: 指定事件，它是epoll_event结构体指针类型

```
struct epoll_event
{
    __uint32_t events;  /*epoll事件*/
    epoll_data_t data;  /*用户数据*/
};
```

events: 描述事件类型。epoll支持的事件类型和poll基本相同。表示epoll事件类型的宏是在poll对应的宏加上“E”. epoll有两个额外的事件类型--EPOLLET和EPOLLONSHOT。他们对于epoll的高效运作非常关键。

data：用于存储用户数据，其类型epoll_data_t

```
typedef union epoll_data
{
    void* ptr;
    int fd;
    uint32_t u32;
    uint64_t u64; 
} epoll_data_t;
```

epoll_data_t是一个联合体，其四个成员使用最多的是fd，它指定事件所从属的目标文件描述符。ptr成员可用来指定与fd相关的用户数据。但由于epoll_data_t是一个联合体，我们不能同时使用其ptr成员和fd成员，所以可以放弃使用epoll_data_t的fd成员，而在ptr指向的用户数据中包含fd。

```
int epoll_wait(int epfd, struct epoll_event* events, int maxevents, int timeout);
```

该函数成功时返回就绪的文件描述符的个数，失败返回-1并设置errno

timeout：与poll接口的timeout参数相同。

maxevents：指定最多监听多少事件，它必须大于0

epoll_wait函数如果检测到事件，就将所有的事件从内核事件表（由epfd参数指定）中复制到它的第二个参数events指向的数组中。这个数组只用于输出epoll_wait检测到的就绪事件，而不像select和poll的数组参数那样即用于传入用户祖册的事件，又用于输出内核检测到的就绪事件。这就极大地提高了应用程序索引就绪文件描述符的效率。

```
//demo
/*如何索引epoll返回的就绪文件描述符*/
int ret = epoll_wait(epollfd, events, MAX_EVENT_MUMBER, -1);
/*仅遍历就绪的ret个文件描述符*/
for(int i = 0; i < ret; i++)
{
    int sockfd = events[i].data.fd;
    /*sockfd肯定就绪，直接处理*/
}
```

* LT和ET模式

epoll对文件描述符的操作有两种模式：LT(Level Trigger，电平触发)模式，ET(EdgeTrigger,边沿触发)模式。LT模式是默认的工作模式，这种模式下epoll相当于一个效率较高的poll。当往epoll内核事件表中注册一个文件描述符上的EPOLLET事件时，epoll将以ET模式来操作该文件描述符。ET模式是epoll的高效工作模式。

对于采用LT工作模式的文件描述符，当epoll_wait检测到其上有事件法师并将此事件通知应用程序后，应用程序可以不立即处理该事件。这样，当应用程序下一次调用epoll_wait时，epoll_wait还会再次向应用程序通告此事件，直到该事件被处理。而对于采用ET工作模式的文件描述符，当epoll_wait检测到其上有事件发生并将此事件通知应用程序后，应用程序必须立即处理该事件，因为后续的epoll_wait调用将不再向应用程序通知这一事件。可见，ET模式在很大程度上降低了同一个epoll事件被重复触发的次数，因此效率要比LT模式高。


* EPOLLONESHOT事件

即使我们使用ET模式，一个socket上的某个事件还是可能被触发多次。这在并发程序中就会引起一个问题。比如一个线程(或进程，下同)在读取完某个socket上的数据后开始处理这些数据，而在数据的处理过程中该socket上又有新数据可读(EPOLLIN在次被触发)，此时另外一个线程被唤醒来读取这些新的数据。于是就出现了两个线程同时操作一个socket的局面。这一点可以使用epoll的EPOLLONESHOT事件实现。

对于注册了EPOLLONESHOT事件的文件描述符，操作系统最多触发其上注册的一个可读，可写或者异常事件，且只触发一次。但是要在线程处理完毕时重置这个socket上的EPOLLONESHOT事件，以确保这个socket下次可读时，其EPOLLIN事件能被触发。


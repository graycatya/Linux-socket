# 高级I/O函数

* 用于创建文件描述符的函数，包括pipe，dup/dup2函数。

```
#include <unistd.h>
int pipe(int fd[2]);
```

pipe: 创建一个管道，可以实现进程间通信。创建了两个文件描述符fd -> fd[0]，fd[1]分别构成管道的两端。 fd[1] 为写端， fd[0]为读端。 默认情况下这一对文件描述符都是阻塞的。Linux2.6.11内核起，管道容量的大小默认为65536字节，可以使用fcntl函数来修改管道容量。 

```
#include <sys/types.h>
#include <sys/socket.h>
int socketpair(int domain, int type, int protocol, int fd[2]);
```

socketpair: 创建双向管道，前三个参数与socket一样

domain: 只能使用UNIX本地域协议族AF_UNIX。

```
#include <unistd.h>
int dup(int file_descriptor);
int dup2(int file_descriptor_one, int file_descriptor_two);
```

dup: 重定向，创建一个新的描述符，该新文件描述符和原有文件描述符file_descriptor指向相同的文件，管道或者网络连接。并且dup返回的文件描述符总是取系统当前可用的最小整数值。dup2和dup类似，不过它将返回第一个不小于file_descriptor的整数值。

* 用于读写数据的函数，包括readv/writev，sendfile，mmap/munmap，splice和tee函数。

```
#include <sys/uio.h>
ssize_t readv(int fd, const struct iovec* vector, int count);
ssize_t writev(int fd, const struct iovec* vector, int count);

struct lovec
{
    void *lov_base; /*内存起始地址*/
    size_t lov_len; /*这块内存的长度*/
};
```

count: vector数组长度

readv: 将数据从文件描述符读到分散的内存块中。
writev: 将多块分散的内存数据一并写入文件描述符中。


* 用于控制I/O行为和属性，包括fcntl函数

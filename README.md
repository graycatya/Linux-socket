# C/C++ socket API

* [1.主机字节序和网络字节序](#1)
* [2.Linux socket地址 结构体](#2)
* [3.Linux ip地址转换函数](#3)
* [4.Linux socket API](#4)

<h2 id="1">1. 主机字节序和网络字节序</h2>
* Linux 提供4个函数来完成主机字节序和网络字节序之间的转换(#include<netinet/in.h>)

1. unsigned long int htonl(unsigned long int hostlong); //主机转网络字节序  ,转4字节 ,小端转大端
2. unsigned short int htons(unsigned short int hostshort); //主机转网络字节序  转2字节,端口大转小
3. unsigned long int ntohl(unsigned long int netlong);  //网络转主机字节序  转4字节，大端转小端
4. unsigned short int ntohs(unsigned short int netshort);   //网络转主机字节序  转2字节,端口大转小
长整型一般用来转换IP地址，短整型一般用来转换端口号。

<h2 id="2">2. Linux socket地址 结构体</h2>

![协议族和地址族](./img/family.png)

* 通用网络结构体

```
//旧的
struct sockaddr
{
    sa_family_t sa_family;
    char sa_data[14];
}
```
sa_family : 地址族类型(sa_family_t)变量。
sa_data成员用于存放socket地址值。不同的协议族的地址值具有不同的含义和长度。

![协议族和地址族](./img/PE.png)

```
//新的
struct sockaddr_storage
{
    sa_family_t sa_family;
    unsigned long int __ss_align;
    char __ss_padding[128-sizeof(__ss_align)];
}
```
__ss_align成员是内存对齐的。


* 专用socket
UNIX本地域协议族使用如下专用socket地址结构体(#include<sys/un.h>)：

```
struct sockaddr_un
{
    sa_family_t sin_family; /*地址族：AF_UNIX*/
    char sun_path[108]; /*文件路径名*/
};
```

TCP/IP协议族有sockaddr_in 和 sockaddr_in6 两个专用socket地址结构体，他们分别用于IPv4 和 IPv6：
```
//IPv4
struct sockaddr_in
{
    sa_family_t sin_family; /*地址族：AF_INET*/
    u_int16_t sin_port; /*端口号，要用网络字节序表示*/
    struct in_addr sinaddr; /*IPv4 地址结构，*/
}
struct in_addr
{
    u_int32_t s_addr; /*IPv4地址， 要用网络字节序表示*/
};

struct sockaddr_in6
{
    sa_family_t sin6_family;    /*地址族: AF_INET6*/
    u_int16_t sin6_port;    /*端口好，要用网络字节序表示*/
    u_int32_t sin6_flowinfo; /*流信息， 应设置为0*/
    struct in6_addr sin6_addr; /*IPv6地址结构体*/
    u_int32_t sin6_scope_id; /*scope ID, 目前处于实验阶段*/
};

struct in6_addr
{
    unsigned char sa_addr[16]; /*IPv6地址， 要用网络字节序表示*/
};
```

所有专用的socket地址(以及sockaddr_storage)类型的变量在实际使用时都需要转化为通用socket地址类型sockaddr。

<h2 id="3">3. Linux ip地址转换函数</h2>

下面3个函数可用于点分十进制字符串表示的IPv4地址和用网络字节序整数表示的IPv4地址间转换(#include<arpa/inet.h>):
1. in_addr_t inet_addr(const char* strptr);
inet_addr： 将点分十进制字符串表示的IPv4地址转化为网络字节序整数表示IPv4地址，它失败时返回INADDR_NONE
2. int inet_aton(const char* cp, struct in_addr* inp);
inet_aton: 和inet_addr功能一样，但是将转化结果存储于参数inp指向的地址结构中，成功返回1，失败返回0
3. char* inet_ntoa(struct in_addr in);
inet_ntoa将用网络字节序整数表示的IPv4地址转化为用点分十进制字符串表示的IPv4地址。（它是一个不可重入函数）
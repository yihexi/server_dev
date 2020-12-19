socket使用系统调用创建，返回一个文件描述符，后续的系统调用都使用这个文件描述符来操作socket。

```c
fd = socket(domain, type, protocol);
```

domain一般有3种类型：`AF_UNIX`, `AF_INET`, `AF_INET6`，分别对应Unix Socket，IPv4 Socket和IPv6Socket。socket是一种进程间通信的解决方案，并不专指网络通信。

type有两种：`SOCK_STREAM`和`SOCK_DGRAM`。这是两种数据传输的格式，stream类型的建立稳定可靠的连接，消息之间没有边界，也就是流式。dgram类型不需要连接，不保证可靠性，有明显的消息边界，也就是报文式。

流式socket流程:

<img src="./img/1.png" style="zoom:50%;" />

报文式流程:

<img src="./img/2.png" style="zoom:50%;" />



在报文上socket上可以connect么？connect会有什么后果？
```c
数据报的发送可以使用write或send，会自动发到对端的socket上
只能读取对端socket的数据
```

ssize_t和size_t的区别？

```c
size_t的真实类型与操作系统有关，在32位架构中被普遍定义为：
typedef   unsigned int size_t;
而在64位架构中被定义为：
typedef  unsigned long size_t;
ssize_t是size_t的有符号版本
```

listen的backlog参数是做什么用的？

```c
int listen(int sockfd, int backlog);
小于backlog数的连接可以立即建立，大于的需要等accept调用，将pending队列中的连接移除。
```

通用的sockaddr怎么使用？

```c
struct sockaddr {
	sa_family_t sa_family; //AF_* 
	char sa_data[14];
}

创建完socket之后需要绑定到一个地址上，通用的函数bind签名如下：
int bind(int sockfd, const struct sockaddr *addr, socklen_t addrlen);
其中第二个参数就是通用的socketaddr类型。
实际使用的时候是要传入一个具体类型的sockaddr的，比如UNIX domain类型的socket使用，先创建一个struct sockaddr_un，
struct sockaddr_un { 
  sa_family_t sun_family; 
  char sun_path[108]; 
};
然后在调用bind的时候强制转换为一个struct sockaddr *
bind(sfd, (struct sockaddr *) &addr, sizeof(struct sockaddr_un))
可以根据addr的第一个字段来判断到底是哪个类型的sockaddr。
//todo：转换的两个函数
```

accept的第二个参数sock_addr是做什么用的？

```c
int accept(int sockfd, struct sockaddr *addr, socklen_t *addrlen);

addr参数用于获取peer socket的地址。如果不感兴趣可以设置为NULL，然后addrlen设置为0

```








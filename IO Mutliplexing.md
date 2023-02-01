# Socket & IO Mutliplexing

## 前言

在面试中经常出现IO复用的问题，然而计算机网络中似乎并没有介绍这三者的理论与实践，因此我花了一些时间，用C写了一个简单的Socket Server与Client，学习这select poll epoll函数的用法与区别以及socket编程。

## 基于TCP Protocol的Socket通信流程

<img src="C:\Users\Young\AppData\Roaming\Typora\typora-user-images\image-20230131210852179.png" alt="image-20230131210852179" style="zoom:67%;" />



## 函数



```c
/*	domain stands for the protocol families, usually set as PF_INET(which is "IP protocol family").
	type specifies the communication semantics, type一般设置为SOCK_STREAM(表示Tcp连接，提供序列化的、可靠的、双向连接的字节流。支持带外数据传输)
	The protocol specifies a particular protocol to be used with the socket. Normally only a single protocol exists to support a particular socket type within a given protocol family, in which case protocol can be specified as 0. 
	
	无论是 Windows 还是 Linux 平台，默认创建的 socket 都是阻塞模式的。  */
extern int socket (int __domain, int __type, int __protocol) __THROW;
```



 ```c
/*	the message is found in buf and has length len.
	flags argument is the bitwise OR of zero or more of the following flags:
MSG_CONFIRM/MSG_DONTROUTE/MSG_DONTWAIT/MSG_EOR/MSG_MORE/MSG_NOSIGNAL/MSG_OOB.	
	if int flags are equal to 0, it means that no flags are specified  */
send(int sockfd, const void *buf, size_t len, int flags);

/*	write to a file descriptor, similar to send(). If the flags argument in send() is set as 0, then send() is equivalent to write()  */
extern ssize_t write (int __fd, const void *__buf, size_t __n);

//a specific example of how to set flags in send
//recv(sockfd, buf, buflen,  MSG_PEEK | MSG_DONTWAIT);
 ```

<img src="https://picx.zhimg.com/80/d36e9cdfd34338179c10ab9c3120bbce_1440w.webp?source=1940ef5c" alt="img" style="zoom: 67%;" />



```c
/*  the meaning of the arguments in recv() can be easily deducted by send(), except flags  */
extern ssize_t recv (int __fd, void *__buf, size_t __n, int __flags);

extern ssize_t read (int __fd, void *__buf, size_t __nbytes)
```

[Where you can find more about flags argument in send and recv](https://stackoverflow.com/questions/24430564/meaning-of-flag-in-socket-send-and-recv)

see also: 

https://man7.org/linux/man-pages/man2/send.2.html

https://man7.org/linux/man-pages/man2/recv.2.html

https://github.com/balloonwj/CppGuide/blob/master/articles/%E7%BD%91%E7%BB%9C%E7%BC%96%E7%A8%8B/socket%E7%9A%84%E9%98%BB%E5%A1%9E%E6%A8%A1%E5%BC%8F%E5%92%8C%E9%9D%9E%E9%98%BB%E5%A1%9E%E6%A8%A1%E5%BC%8F.md

### Client End

```c
/*	fd stands for file descriptor of the socket, in Linux "everything is a file", including socket connections
	sockaddr is a structure describing a generic socket address. 
	The addrlen argument specifies the size of addr.  */
extern int connect (int __fd, __CONST_SOCKADDR_ARG __addr, socklen_t __len)
//# define __CONST_SOCKADDR_ARG	const struct sockaddr *
```



### Server End

```c
/*	Give the socket FD the local address ADDR (which is LEN bytes long).
	that is to say bind the address to a specific socket.
	addrlen specifies the size(in bytes) of the address structure pointed to by addr,
	which is sizeof*(__addr)  */
extern int bind (int __fd, __CONST_SOCKADDR_ARG __addr, socklen_t __len) __THROW;

```

```c
/*	Prepare to accept connections on socket FD.
	N connection requests will be queued before further requests are refused.  */
extern int listen (int __fd, int __n) __THROW;
```

> more about the meaning of n : "在进程正理一个一个连接请求的时候，可能还存在其它的连接请求。因为TCP连接是一个过程，所以可能存在一种半连接的状态，有时由于同时尝试连接的用户过多，使得服务器进程无法快速地完成连接请求。如果这个情况出现了，服务器进程希望内核如何处理呢？内核会在自己的进程空间里维护一个队列以跟踪这些完成的连接但服务器进程还没有接手处理或正在进行的连接"

```c
/* Await a connection on socket FD.
   When a connection arrives, open a new socket to communicate with it,
   set *ADDR (which is *ADDR_LEN bytes long) to the address of the connecting
   peer and *ADDR_LEN to the address's actual length, and return the
   new socket's descriptor, or -1 for errors.
 */
extern int accept (int __fd, __SOCKADDR_ARG __addr, socklen_t *__restrict __addr_len);
```

```c
/*  Close the file descriptor FD.  */
extern int close (int __fd);
```

### select poll epoll

```c
int select(int nfds,
            fd_set *restrict readfds,
            fd_set *restrict writefds,
            fd_set *restrict errorfds,
            struct timeval *restrict timeout);
```

Disadvantage: 

- The biggest drawback of select is that there is a limit to the number of fd’s that can be opened by a single process, which is set by FD_SETSIZE; the default value is 1024;
- Each time select is called, the set of fd’s needs to be copied from user state to kernel state, which is a significant overhead when there are a lot of fd’s;
- Each kernel needs to scan the entire fd_set linearly, so its I/O performance decreases linearly as the number of descriptor fd’s monitored grows;
- 



`select` 和 `poll` 监听文件描述符list，进行一个线性的查找 O(n)

`epoll`: 使用了内核文件级别的回调机制O(1)

epoll_create函数：

```
int epoll_create(int size);
```

创建一个 epoll 对象，返回该对象的描述符，注意要使用 close 关闭该描述符。



epoll_ctl函数：

```
int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event)
```

op: op stands for operation code, EPOLL_CTL_ADD，EPOLL_CTL_DEL，EPOLL_CTL_MOD

fd: 需要添加监听的 socket 描述符

event: 事件信息

(to be completed)

## code demo

to be upload
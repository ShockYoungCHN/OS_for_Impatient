# Socket & IO Mutliplexing

## 前言

在面试中经常出现IO复用的问题，然而计算机网络中似乎并没有介绍这三者的理论与实践，因此我花了一些时间，用C写了一个简单的Socket Server与Client，学习同步/异步、阻塞/非阻塞、socket编程、select poll epoll实例的用法与区别。

## 基于TCP Protocol的Socket通信流程

<img src=".\md_image\process_of_socket.png" style="zoom:67%;" />



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



关于Sync/Async, Blocking/Non-Blocking，实在有太多版本的解读，在不同语境/层次下（比如Nodejs并发和Linux I/O），他们所指的东西也不同，甚至出现混用的情况；因此，在这里我只从Linux IO模型的角度讨论Sync/Async, Blocking/Non-Blocking。

## 同步与异步

同步与异步关注的是任务完成时消息通知的方式。以下对于同步和异步的定义是我的个人理解，欢迎指正。

同步：调用IO后，函数返回和IO结果同时发生。

异步：调用IO后，函数直接返回一个值，但是IO的结果会在一段时间后得到。

>引用：
>
>POSIX defines these two terms as follows:
>
>- A synchronous I/O operation causes the requesting process to be blocked until that I/O operation completes.
>- An asynchronous I/O operation does not cause the requesting process to be blocked.
>
>用符号表达就是：
>
>A synchronous I/O operation -> blocked
>
>An asynchronous I/O -> non blocked
>
>但是我认为POSIX的这两句话并不算definition，只是说明了同步和异步的一些properties。



<img src="https://picx.zhimg.com/80/v2-e0180a5ffebd91c480d0ccdc02c6d2a7_1440w.webp?source=1940ef5c" alt="img" style="zoom:67%;" />



### IO Model实例

**异步IO Model(AIO)：**

<img src=".\md_image\image-20230202204213360.png" alt="image-20230202204213360" style="zoom:67%;" />

another picture describe AIO：

<img src="https://ask.qcloudimg.com/http-save/yehe-7686797/brkn4tkcm2.png?imageView2/2/w/1620" alt="img" style="zoom:67%;" />



如上图，显然这是一个异步的IO Model，而且他也是一个non blocking model，因为调用aio_read后立刻返回。



**IO复用(IO Multiplexing): **

IO复用一定是阻塞的，

但是关于IO复用是同步还是异步一直是一个有争议的话题，因此借用知乎一篇回答的说法：

>- select/poll/epoll 这个系统调用，是同步的，也就是必须等待操作系统返回值。
>- 而底层用了select/poll/epoll 的封装后的框架，可以是异步的，只要你暴露给外部的接口无需等待你的返回值即可。
>
>[IO多路复用到底是不是异步的](https://www.zhihu.com/question/59975081/answer/1932776593)



<img src=".\md_image\image-20230202220714438.png" alt="image-20230202220714438" style="zoom: 67%;" />

## 阻塞与非阻塞

阻塞与非阻塞关注的是接口调用（发出请求）后等待数据返回时的状态。

阻塞：被挂起无法执行其他操作的则是阻塞型的

非阻塞：可以被立即「抽离」去完成其他「任务」的则是非阻塞型的

<img src="https://pic3.zhimg.com/80/v2-6507ab3517814b1b84fbff9a3eb31842_1440w.webp" alt="img" style="zoom:67%;" />

### IO Model实例

以下两种IO Model都是同步的IO Model，因为他们都造成了阻塞，只是阻塞程度不同。

**阻塞的IO Model：**

在内核中no data ready的情况下调用read函数，read函数会一直等待，直到内核中data ready，然后拷贝到用户区。这种做法的缺点显而易见，如果一直没有data的话，进程会在原地等待，无法做其他事情。

<img src=".\md_image\image-20230202202516674.png" alt="image-20230202202516674" style="zoom:67%;" />



**非阻塞的IO Model：**

每次在内核中no data ready的情况下调用read函数，都会直接返回EWOULDBLOCK，但是在内核中data ready后，就会花费一段时间，把data从内核区中拷贝到用户区。

<img src=".\md_image\image-20230202202440760.png" alt="image-20230202202440760" style="zoom:67%;" />



## 关系

“Note that synchronous/asynchronous implies blocking/not blocking but not vice versa, that is, not every blocking operation is synchronous and not every non blocking operation is asynchronous.” 

同步/异步与阻塞/非阻塞之间没有必然关系，可以任意组合。另有下图供参考。





![图片描述](https://segmentfault.com/img/bVbcRPz?w=386&h=225)

>一点个人的建议：不要过分纠结于IO复用到底是同步还是异步，关键是理解各个IO模型是为了解决其他IO模型的什么问题而提出的。建议阅读：[IO多路复用到底是不是异步的](https://www.zhihu.com/question/59975081/answer/1932776593)

以上提出了4中IO模型并作出了简单的介绍，还有一种Signal Driven I/O Model没有提到，我会后期补充。下文会着重介绍I/O Multiplexing。

除了以上链接，还有一些**有价值**的阅读材料：

https://notes.shichao.io/unp/ch6/#io-multiplexing-model

https://www.cs.toronto.edu/~krueger/csc209h/f05/lectures/Week11-Select.pdf

[Synchronous vs Asynchronous (unc.edu)](https://www.cs.unc.edu/~dewan/242/s07/notes/ipc/node9.html#:~:text=A synchronous operation blocks a process till the,what it means for an operation to complete.)



## I/O Multiplexing

Definition: I/O multiplexing is the capability to tell the kernel that we want to be notified if one or more I/O conditions are ready, like input is ready to be read, or descriptor is capable of taking more output. 

我认为这句话的本质在“tell the kernel”，意思是让kernel去遍历这些fd并告诉我们哪些是满足条件的。因为用户本身也可以去一遍遍的system call去检查哪些fd满足条件；但是system call会触发用户态和内核态的切换，这样非常耗费时间。在IO Multiplexing中，只需要一次调用system call，就能得到全部信息。

> multiplexing的本意是: **multiplexing** (sometimes contracted to **muxing**) is a method by which multiple analog or digital signals are combined into one signal over a [shared medium](https://en.wikipedia.org/wiki/Shared_medium).

### select poll epoll

#### select

```c
/*
After select() has returned, readfds/writefds will be cleared of all file descriptors except for those that ready for reading/writing, and errorfds will be cleared of all file descriptors except for those which an exceptional condition has occurred.
简而言之，readfds/writefds/errorfds中除了满足读/写/报错条件的fd以外的fd都被剔除了。

return:
On success, select() return the number of file descriptors contained in the three returned descriptor sets("these three returned descriptor sets"就是readfds/writefds/errorfds)
*/
int select(int nfds,
            fd_set *restrict readfds,
            fd_set *restrict writefds,
            fd_set *restrict errorfds,
            struct timeval *restrict timeout);
```

```c
       //Arguments 
       readfds
              The file descriptors in this set are watched to see if
              they are ready for reading.  A file descriptor is ready
              for reading if a read operation will not block; in
              particular, a file descriptor is also ready on end-of-
              file.

              After select() has returned, readfds will be cleared of
              all file descriptors except for those that are ready for
              reading.

       writefds
              The file descriptors in this set are watched to see if
              they are ready for writing.  A file descriptor is ready
              for writing if a write operation will not block.  However,
              even if a file descriptor indicates as writable, a large
              write may still block.

              After select() has returned, writefds will be cleared of
              all file descriptors except for those that are ready for
              writing.

       exceptfds
              The file descriptors in this set are watched for
              "exceptional conditions".  For examples of some
              exceptional conditions, see the discussion of POLLPRI in
              poll(2).

              After select() has returned, exceptfds will be cleared of
              all file descriptors except for those for which an
              exceptional condition has occurred.

       nfds   This argument should be set to the highest-numbered file
              descriptor in any of the three sets, plus 1.  The
              indicated file descriptors in each set are checked, up to
              this limit (but see BUGS).

       timeout
              The timeout argument is a timeval structure (shown below)
              that specifies the interval that select() should block
              waiting for a file descriptor to become ready.  The call
              will block until either:

              • a file descriptor becomes ready;

              • the call is interrupted by a signal handler; or

              • the timeout expires.

              Note that the timeout interval will be rounded up to the
              system clock granularity, and kernel scheduling delays
              mean that the blocking interval may overrun by a small
              amount.

              If both fields of the timeval structure are zero, then
              select() returns immediately.  (This is useful for
              polling.)

              If timeout is specified as NULL, select() blocks
              indefinitely waiting for a file descriptor to become
              ready.
/*          
    struct timeval {
        time_t      tv_sec;         /* seconds */
        suseconds_t tv_usec;        /* microseconds */
    };
*/
```

> More about timeout
>
> There are 3 situations for timeout:
>
> **Wait forever** (timeout is specified as a null pointer) until one of the specified descriptors is ready for I/O.
>
> **Wait up to a fixed amount of time** (timeout points to a timeval structure). Return when one of the specified descriptors is ready for I/O, but do not wait beyond the number of seconds and microseconds specified in the timeval structure.
>
> **Do not wait at all** (timeout points to a timeval structure and the timer value is 0, i.e. the number of seconds and microseconds specified by the structure are 0). Return immediately after checking the descriptors. This is called polling.

Advantage:

Copying readfds, writefds, errorfds to kernel space and traverse these fd_set in kernel to check if the file descriptors become "ready" for some class of I/O operation. **Only accessing kernel space for one time, all the file descriptor can be monitored.**

Disadvantage: 

- The biggest drawback of select is that there is a <span id="limit" style="color:#F00">limit</span> to the number of fd’s that can be opened by a single process, which is set by FD_SETSIZE; the default value is 1024;
- Each time select is called, the set of fd’s needs to be copied from user state to kernel state, which is a significant overhead when there are a lot of fd’s;
- Each kernel needs to scan the entire fd_set linearly, so its I/O performance decreases linearly as the number of descriptor fd’s monitored grows;

> see also:
>
> [Chapter 6. I/O Multiplexing: The select and poll Functions - Shichao's Notes](https://notes.shichao.io/unp/ch6/#select-function)
>
> [select(2) - Linux manual page (man7.org)](https://www.man7.org/linux/man-pages/man2/select.2.html)



##### demo

(to be completed

#### poll

poll() is almost the same as select(). Poll passes file descriptors as **arrays in the user state** and stores them as **linked list in the kernel**, the number of fds has <a href="#limit">no maximum limit</a>( this is very different from select() ).

```c
/*
fds is a pointer pointing to arrays
the number of items in the fds array is specified in nfds
timeout is the same meaning as in select()

return:        
On success, poll() returns a nonnegative value which is the number of elements in the pollfds whose revents fields have been set to a nonzero value (indicating an event or an error).  A  return value of zero indicates that the system call timed out before any file descriptors became read.

On error, -1 is returned, and errno is set to indicate the error.
*/
int poll(struct pollfd *fds, nfds_t nfds, int timeout);

/*
struct pollfd {
    int   fd;         /* file descriptor */
    short events;     /* requested events */
    short revents;    /* returned events */
};
*/
```

Disadvantage:

High performance overhead because the kernel still have to traverse all the fd (almost the same overhead as select() )

##### demo

(to be completed)

#### epoll

`epoll` is a improvement for `poll()` and `select()`

>Storing a collection of file descriptors using a <a href="#why_rb_tree">red-black tree</a> instead of a list

`epoll` model is different from `poll` and `select` model, to use `epoll` model, there are 3 functions: `epoll_create`, `epoll_ctl`,  `epoll_wait`.

##### epoll_create()

to create an epoll instance, you may first use `epoll_create()`

```c
/*
size: Since Linux 2.6.8, the size argument is ignored, but must be greater than zero.
return 0 for create epoll successfully, -1 for failed
*/
int epoll_create(int size);
```

##### epoll_ctl()

then using `epoll_ctl()` to add/delete/modify fd in the interesting list. `ctl` stand for control.

```c
/*
epfd: the fd of epoll instance
op: op stands for operation code, which is EPOLL_CTL_ADD, EPOLL_CTL_DEL, EPOLL_CTL_MOD
fd: the file decriptor that to be added/deleted/modified in the interesting list

return: When successful, epoll_ctl() returns zero.  When an error occurs, epoll_ctl() returns -1 and errno is set to indicate the error.
*/
int epoll_ctl(int epfd, int op, int fd, struct epoll_event* event);

/*
typedef union epoll_data {
	void    *ptr;
	int      fd;
	uint32_t u32;
	uint64_t u64;
} epoll_data_t;

struct epoll_event {
	uint32_t     events;    /* Epoll events */
	epoll_data_t data;      /* User data variable */
};
*/
```

>when to use EPOLL_CTL_MOD?
>
>**suppose** fd_1 is added into the interesting list by epoll_ctl, then in epoll_wait() fd_1 is found to be ready for read.
>
>**then** the content inside event will be cleared but our fd_1 keeps existed in the insteresting list. 
>
>this means if we want to further more monitor fd_1, **we have to modify fd_1's bundled event**
>
>**Conclusion**:
>
>When fd is existed in the insteresting list, the using `epoll_ctl(epfd,EPOLL_CTL_ADD, fd_1, event)` is incorrect, changeing`EPOLL_CTL_ADD` into `EPOLL_CTL_MOD` should be fine.

##### epoll_wait()

`epoll_wait` is similar to `select`, The epoll_wait() system call waits for events happened in fds in the interesting list.

```c
/*
epfd: the fd of epoll instance
events: in epoll_ctl we probably add some fd into a list known as "the interest list". The buffer pointed to by events is used to return information of the ready file descriptors in the interest list. 
maxevents: Up to maxevents are returned by epoll_wait() and maxevents argument must be greater than zero.

return: 
On success, epoll_wait() returns the number of file descriptors ready for the requested I/O, or zero if no file descriptor became ready during the requested timeout milliseconds.  On failure,  epoll_wait() returns -1 and errno is set to indicate the error.
*/
int epoll_wait(int epfd, struct epoll_event *events, int maxevents, int timeout);
```



##### <span id="why_rb_tree">Why using RB Tree in epoll?</span>

(to be completed)

#### 区别

`select` 和 `poll` 监听文件描述符list，进行一个线性的查找 O(n)

`epoll`: 使用了内核文件级别的回调机制O(1)

## code demo

(to be upload)

# 第七章

同步IO：必须等IO操作完成后，进程才能进行其他操作；

异步IO：无须等IO操作完成，进程就能继续进行其他操作。

**文件描述符：**

当程序打开一个现有文件或者创建一个新文件时，内核向进程返回一个文件描述符。在程序设计中，一些涉及底层的程序编写往往会围绕着文件描述符展开。但是文件描述符这一概念往往只适用于UNIX、Linux这样的操作系统。
**缓存I/O：**

在Linux的缓存I/O机制中，数据会先被拷贝到操作系统内核的缓冲区中，然后才会从操作系统内核的缓冲区拷贝到应用程序的地址空间。下文中的等待数据准备好的意思是等待数据拷贝到内核缓存区中，然后才能进行下一步的数据拷贝（从内核的缓冲区拷贝到用户进程内存中）


## 7.1 四种网络IO模型

- 阻塞IO模型
- 非阻塞IO模型
- 多路IO复用模型
- 异步IO模型

### 阻塞IO模型

Linux中，默认情况下所有socket都是阻塞的。

在阻塞IO模型中，进程一旦调用IO操作后，马上阻塞，直到IO操作（等待数据+拷贝数据）完成后才解除阻塞。

如果要让服务端同时服务多个客户端，就需要用多线程，可能用到“线程池”和“连接池”。

- 线程池是指维持一定合理数量的线程，并让空闲的线程重新承担新的执行任务，降低创建和销毁线程的频率。
- 连接池是指维持连接的缓存池，尽量重用已有的连接，降低创建和关闭连接的频率。

阻塞IO模型中，一个线程只能处理一个IO请求，开销很大，面对大量服务请求时，多线程也会遇到瓶颈，可以考虑使用非阻塞模型。



### 非阻塞IO模型

在非阻塞IO模型中，用户进程需要不断的主动询问内核，数据是否准备好。

比如当用户进程调用read操作时，如果内核中数据未准备好，那么它并不会阻塞进程，而是直接返回一个错误。然后继续重复的调用read操作，直到内核中的数据准备好了，此时内核再收到用户进程的read操作请求，就会将数据复制到用户内存中，然后返回正确的返回值。

注意非阻塞IO模型是使用循环来一直调用IO操作的命令，大幅度占用CPU资源，实际上效率也很低下。而select多路复用模式能够一次检测多个连接是否活跃，效率更高。



### 多路IO复用模型

在多路IO复用模型中，有个函数（select/poll/epoll）会不断地轮训它负责的所有socket，当某个socket数据到达了，就通知用户进程，此时用户进程再调用read操作，将数据拷贝到用户进程的内存中。

值得注意的是，用户进程执行select函数时是阻塞的，用户进程将数据复制到进程内存时也是阻塞的。因此select处理单个连接时没有优势，而处理多个连接时优势明显，因为可以用一个进程处理多个连接。

总的来说，多路IO复用模型实际上也是阻塞的，但它是被select函数阻塞，而不是被单个socket IO阻塞。



### 异步IO模型

在异步IO模型中，用户进程发起read操作后，马上就去做处理其他任务了，在此期间，内核会 等数据准备好，然后将数据拷贝到用户内存中，最后给用户进程发信号，返回read操作已完成的信息。在此过程中，用户进程完全没有被阻塞。

非阻塞IO模型与异步IO模型的区别：

- 非阻塞IO模型不会阻塞用户进程，但需要用户进程一直主动查询。实际上效率很低。
- 异步IO模型不阻塞用户进程，而且用户进程将整个IO操作交给内核完成，用户进程不需要检查IO操作的状态，不需要主动拷贝数据，效率很高。



## 7.2~7.4 多路IO复用模型中的select/poll/epoll

参考[IO多路复用的三种机制Select，Poll，Epoll](https://www.jianshu.com/p/397449cadc9a)

select、poll、epoll都是多路IO复用的机制，即同时监督多个文件描述符，一旦某个描述符就绪（读就绪or写就绪），就通知用户进程进行读写操作。它们都是同步IO，因为它们都需要在读写事件就绪后自己负责读写，即读写过程是阻塞的。

select、poll、epoll机制是依次出现的，后者比前者更完善，性能更优越。

### 1、select

```cpp
int select(int maxfdp1,fd_set *readset,fd_set *writeset,fd_set *exceptset,const struct timeval *timeout);
```

**【参数说明】**
 **int maxfdp1：** 待测试的文件描述字个数。
 **fd_set \*readset , fd_set \*writeset , fd_set \*exceptset：**
 `fd_set`可以理解为一个集合，这个集合中存放的是文件描述符(file descriptor)，即文件句柄。中间的三个参数指定我们要让内核测试读、写和异常条件的文件描述符集合。（如果对某一个的条件不感兴趣，就可以把它设为空指针）
 **const struct timeval \*timeout：** `timeout`告知内核等待所指定文件描述符集合中的任何一个就绪可花多少时间。

**【返回值】int** 

若有就绪描述符返回其数目，若超时则为0，若出错则为-1

**select运行机制**

select()的机制中提供一种`fd_set`的数据结构，该结构体可以看作是一个描述符的集合，可以将fd_set看作是一个位图，其中每个整数的每一bit代表一个描述符。既定义FD_SETSIZE为1024，一个整数占4个字节，既32位，那么就是用包含32个元素的整数数组来表示文件描述符集

fd_set有四个关联的api

```cpp
void FD_ZERO(fd_set *fdset) //清空fdset，将所有bit置为0
void FD_SET(int fd, fd_set *fdset) //将fd对应的bit置为1
void FD_CLR(int fd, fd_set *fdset) //将fd对应的bit置为0
void FD_ISSET(int fd, fd_set *fdset) //判断fd对应的bit是否为1,也就是fd是否就绪
```

当调用select()时，由内核根据IO状态修改fd_set的内容，由此来通知执行了select()的进程哪一Socket或文件可读。

**select优点**

select的优势是用户可以在一个线程内同时处理多个socket的IO请求。用户可以注册多个socket，然后不断地调用select读取被激活的socket，即可达到在同一个线程内同时处理多个IO请求的目的。而在同步阻塞模型中，必须通过多线程的方式才能达到这个目的。

**select缺点**

1. 每次调用select，都需要把`fd_set`集合从用户态拷贝到内核态，fd_set`集合很大时，这个开销也很大。
2. 同时每次调用select都需要在内核遍历传递进来的所有`fd_set`，`fd_set`集合很大时，这个开销也很大。
3. 为了减少数据拷贝带来的性能损坏，内核对被监控的`fd_set`集合大小做了限制(1024)。

### 2、poll

相比select，poll没有最大文件描述符数量的限制，其他机制与select相同。

```cpp
int poll(struct pollfd *fds, nfds_t nfds, int timeout);

typedef struct pollfd {
        int fd;                         // 需要被检测或选择的文件描述符
        short events;                   // 对文件描述符fd上感兴趣的事件
        short revents;                  // 文件描述符fd上当前实际发生的事件
} pollfd_t;
```

poll改变了文件描述符集合的描述方式，使用了`pollfd`结构而不是select的`fd_set`结构，使得poll支持的文件描述符集合限制远大于select的1024。

### 3、epoll

相对于select来说，epoll没有描述符个数限制，使用一个epoll文件描述符管理多个socket文件描述符。

Linux中提供的epoll相关函数如下：

```csharp
int epoll_create(int size);
int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event);
int epoll_wait(int epfd, struct epoll_event * events, int maxevents, int timeout);
```

**1. epoll_create** 函数创建一个epoll句柄，参数`size`表明内核要监听的描述符数量。调用成功时返回一个epoll句柄描述符，失败时返回-1。

**2. epoll_ctl** 函数注册要监听的事件类型。四个参数解释如下：

`epfd` 表示epoll句柄

`op` 表示fd操作类型，有如下3种 

- EPOLL_CTL_ADD 注册新的fd到epfd中
- EPOLL_CTL_MOD 修改已注册的fd的监听事件
- EPOLL_CTL_DEL 从epfd中删除一个fd

 `fd` 是要监听的描述符

 `event` 表示要监听的事件

**3. epoll_wait** 函数等待事件的就绪，成功时返回就绪的事件数目



**epoll优点**

epoll能显著提高程序在大量并发连接中只有少量活跃的情况下的系统CPU利用率。原因就是获取事件的时候，它无须遍历整个被侦听的描述符集，只要遍历那些被内核IO事件异步唤醒而加入Ready队列的描述符集合就行了。

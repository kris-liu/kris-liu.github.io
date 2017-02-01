---
title: I/O 多路复用之select、poll、epoll实现原理及对比总结
date: 2016-12-24 14:00:00
categories: IO&NIO
tags: [IO, NIO]
---
select，poll，epoll都是IO多路复用的机制。I/O多路复用就是通过一种机制，一个进程可以监视多个描述符，一旦某个描述符就绪（一般是读就绪或者写就绪），能够通知程序进行相应的读写操作。但select，poll，epoll本质上都是同步I/O，因为他们都需要在读写事件就绪后自己负责进行读写，也就是说这个读写过程是阻塞的，而异步I/O则无需自己负责进行读写，异步I/O的实现会负责把数据从内核拷贝到用户空间。

<!--more-->

## select

```
int select (int n, fd_set *readfds, fd_set *writefds, fd_set *exceptfds, struct timeval *timeout);
```

select函数监视的文件描述符分3类，分别是writefds、readfds、和exceptfds。调用后select函数会阻塞，直到有描述副就绪（有数据 可读、可写、或者有except），或者超时（timeout指定等待时间，如果立即返回设为null即可），函数返回。当select函数返回后，可以通过遍历fdset，来找到就绪的描述符。

### select的缺点：

1. select支持的文件描述符数量太小了，默认是1024

2. 每次调用select都需要把fd集合从用户态拷贝到内核态，这个开销在fd很多时会很大

3. 每次调用select都需要在内核遍历传递进来的所有fd，查看有没有就绪的fd，这个开销在fd很多时也很大，效率随FD数目增加而线性下降

## poll

```
int poll (struct pollfd *fds, unsigned int nfds, int timeout);
```

不同与select使用三个位图来表示三个fdset的方式，poll使用一个 pollfd的指针实现。

```
struct pollfd {
    int fd; /* file descriptor */
    short events; /* requested events to watch */
    short revents; /* returned events witnessed */
};
```

pollfd结构包含了要监视的event和发生的event，不再使用select“参数-值”传递的方式。同时，pollfd并没有最大数量限制（但是数量过大后性能也是会下降）。 和select函数一样，poll返回后，需要轮询pollfd来获取就绪的描述符。

从上面看，select和poll都需要在返回后，通过遍历文件描述符来获取已经就绪的socket。事实上，同时连接的大量客户端在一时刻可能只有很少的处于就绪状态，因此随着监视的描述符数量的增长，其效率也会线性下降。

poll还有一个特点是“水平触发”，如果报告了fd后，没有被处理，那么下次poll时会再次报告该fd。

### poll的缺点：

1. 每次调用select都需要把fd集合从用户态拷贝到内核态，这个开销在fd很多时会很大

2. 每次调用select都需要在内核遍历传递进来的所有fd，查看有没有就绪的fd，这个开销在fd很多时也很大，效率随FD数目增加而线性下降

### poll跟select对比的优点

1. 解决了select的第一个缺点，poll没有最大文件描述符数量的限制

## epoll

epoll是在2.6内核中提出的，是之前的select和poll的增强版本。相对于select和poll来说，epoll更加灵活，没有描述符限制。epoll使用一个文件描述符管理多个描述符，将用户关心的文件描述符的事件存放到内核的一个事件表中，这样在用户空间和内核空间的copy只需一次。

epoll显著提高程序在大量并发连接中只有少量活跃的情况下的系统CPU利用率，因为它会复用文件描述符集合来传递结果而不用迫使开发者每次等待事件之前都必须重新准备要被监听的文件描述符集合，另一点原因就是获取事件的时候，它无须遍历整个被侦听的描述符集，只要遍历那些被内核IO事件异步唤醒而加入就绪队列的描述符集合就行了。

### epoll操作过程

epoll操作过程需要三个接口，分别如下：

```
int epoll_create(int size)；
int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event)；
int epoll_wait(int epfd, struct epoll_event * events, int maxevents, int timeout);
```

1. `int epoll_create(int size);`
创建一个epoll的句柄，参数size并不是限制了epoll所能监听的描述符最大个数，只是对内核初始分配内部数据结构的一个建议。
当创建好epoll句柄后，它就会占用一个fd值，在linux下如果查看/proc/进程id/fd/，是能够看到这个fd的，所以在使用完epoll后，必须调用close()关闭，否则可能导致fd被耗尽。

2. `int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event)；`
函数是对指定描述符fd执行op操作。
 - epfd：是epoll_create()的返回值。
 - op：表示op操作，用三个宏来表示：添加EPOLL_CTL_ADD，删除EPOLL_CTL_DEL，修改EPOLL_CTL_MOD。分别添加、删除和修改对fd的监听事件。
 - fd：是需要监听的fd（文件描述符）
 - epoll_event：是告诉内核需要监听什么事，struct epoll_event结构如下：

```
struct epoll_event {
  __uint32_t events;  /* Epoll events */
  epoll_data_t data;  /* User data variable */
};

//events可以是以下几个宏的集合：
EPOLLIN ：表示对应的文件描述符可以读（包括对端SOCKET正常关闭）；
EPOLLOUT：表示对应的文件描述符可以写；
EPOLLPRI：表示对应的文件描述符有紧急的数据可读（这里应该表示有带外数据到来）；
EPOLLERR：表示对应的文件描述符发生错误；
EPOLLHUP：表示对应的文件描述符被挂断；
EPOLLET： 将EPOLL设为边缘触发(Edge Triggered)模式，这是相对于水平触发(Level Triggered)来说的。
EPOLLONESHOT：只监听一次事件，当监听完这次事件之后，如果还需要继续监听这个socket的话，需要再次把这个socket加入到EPOLL队列里
```

3. `int epoll_wait(int epfd, struct epoll_event * events, int maxevents, int timeout);`
等待epfd上的io事件，最多返回maxevents个事件。
参数events用来从内核得到事件的集合，maxevents告之内核这个events有多大，这个maxevents的值不能大于创建epoll_create()时的size，参数timeout是超时时间（毫秒，0会立即返回，-1将永久阻塞直到有监听的io事件发生）。该函数返回需要处理的事件数目，如返回0表示已超时。

### epoll工作模式:

epoll对文件描述符的操作有两种模式：LT（level trigger）和ET（edge trigger）。LT模式是默认模式，LT模式与ET模式的区别如下：

 - LT模式：当epoll_wait检测到描述符事件发生并将此事件通知应用程序，应用程序可以不立即处理该事件。下次调用epoll_wait时，会再次响应应用程序并通知此事件。

 - ET模式：当epoll_wait检测到描述符事件发生并将此事件通知应用程序，应用程序必须立即处理该事件。如果不处理，下次调用epoll_wait时，不会再次响应应用程序并通知此事件。

### epoll的跟select和poll对比的优点：

1. 支持一个进程打开大数目的socket描述符。它所支持的FD上限是最大可以打开文件的数目，这个数字一般远远于2048，具体数目可以cat /proc/sys/fs/file-max察看,一般来说这个数目和系统内存关系很大。

2. select和poll每次调用都需要把所有要监听的fd重新拷贝到内核空间；epoll只在调用epoll_ctl时拷贝一次要监听的fd，调用epoll_wait时不需要每次把所有要监听的fd重复拷贝到内核空间。

3. IO效率不随FD数目增加而线性下降。传统的select/poll另一个致命弱点就是当你拥有一个很大的socket集合，任一时间只有部分的socket是"活跃"的，但是select/poll每次调用都会线性扫描全部的集合，导致效率呈现线性下降。但是epoll不存在这个问题，它只会对"活跃"的socket进行操作。这是因为在内核实现中epoll是根据每个fd上面的callback函数实现的。那么，只有"活跃"的socket才会主动的去调用callback函数，其他idle状态socket则不会。


## 总结

### Select、Poll与Epoll区别

|     | Select | Poll | Epoll |
|:---:|:------:|:----:|:-----:|
|支持最大连接数|1024（x86） or 2048（x64）|无上限|无上限|
|IO效率|每次调用进行线性遍历，时间复杂度为O（N）|每次调用进行线性遍历，时间复杂度为O（N）| 使用“事件”通知方式，每当fd就绪，系统注册的回调函数就会被调用，将就绪fd放到rdllist里面，这样epoll_wait返回的时候我们就拿到了就绪的fd。时间发复杂度O（1）|
|fd拷贝|每次select都拷贝|每次poll都拷贝|调用epoll_ctl时拷贝进内核并由内核保存，之后每次epoll_wait不拷贝|


1. select有最大文件描述符数量限制；poll，epoll则没有最大文件描述符数量限制。

2. select，poll每次调用都要把fd集合从用户态往内核态拷贝一次，并且要把current往设备等待队列中挂一次，而epoll只在调用epoll_ctl时拷贝一次，而且把current往等待队列上挂也只挂一次（在epoll_wait的开始，注意这里的等待队列并不是设备等待队列，只是一个epoll内部定义的等待队列）。这也能节省不少的开销。

3. select，poll实现需要自己不断轮询所有fd集合，直到设备就绪，期间可能要睡眠和唤醒多次交替。而epoll其实也需要调用epoll_wait不断轮询就绪链表，期间也可能多次睡眠和唤醒交替，但是它是设备就绪时，调用回调函数，把就绪fd放入就绪链表中，并唤醒在epoll_wait中进入睡眠的进程。虽然都要睡眠和交替，但是select和poll在“醒着”的时候要遍历整个fd集合，而epoll在“醒着”的时候只要判断一下就绪链表是否为空就行了，这节省了大量的CPU时间。这就是回调机制带来的性能提升。

4. 如果没有大量的idle-connection或者dead-connection，epoll的效率并不会比select/poll高很多，但是当遇到大量的idle-connection，就会发现epoll的效率大大高于select/poll。


## 参考资料

[Linux IO模式及 select、poll、epoll详解](https://segmentfault.com/a/1190000003063859)  

[select、poll、epoll之间的区别总结](http://www.cnblogs.com/Anker/p/3265058.html)  

[select,poll,epoll的归纳总结区分](http://blog.csdn.net/turkeyzhou/article/details/8504554)

---
layout: post
category: "Linux"
title:  "浅析几种网络I/O模型"
tags: [Linux]
---

# UNIX网络编程读书笔记

## （一）网络I/O中的阻塞、非阻塞、同步和异步

在网络编程中，阻塞、非阻塞、同步和异步这些名词经常被提到，是深入学习和理解网络编程的基础。下面结合《UNIX网络编程》和自己的理解，作个小结。

网络I/O(*network I/O*)过程涉及两个对象，一个是调用这个IO操作的进程(*process*)或线程(*thread*)，另一个是操作系统（这里指Linux操作系统）内核(*kernel*)。以*read*操作为例，这个操作具体可以分为两个过程：

1. 等待数据可读(Waiting for the data to be ready)
2. 将数据从内核拷贝到进程中(Copying the data from the kernel to the process)

各种网络I/O模型的区别就在于这两个过程。具体来说，网络I/O模型可以分为阻塞(*blocking I/O*)，非阻塞(*nonblocking I/O*)，I/O多路复用(*I/O multiplexing*)，信号驱动(*signal driven I/O*)和异步(*asynchronous I/O*)这几种。下面详细介绍这几种网络I/O模型。

### blocking I/O（阻塞I/O）
在Linux系统中，默认情况下socket为阻塞的，读操作流程如下：
![](http://images.cnitblog.com/i/434101/201406/281848093983364.jpg)
当用户进程application调用recvfrom这个系统调用时，由于kernel此时没有准备好数据，因此会一直等待数据，用户进程阻塞在recvfrom调用中，直到有数据到达，kernel将数据从内核拷贝到内存中，拷贝完成后返回，用户进程的阻塞结束。所以，blocking I/O在I/O操作的两个阶段都是阻塞的。


### non-blocking I/O（非阻塞I/O）
如果将socket设置为非阻塞的，读操作流程如下：
![](http://images.cnitblog.com/i/434101/201406/281849563679542.jpg)
当用户进程调用recvfrom时，如果kernel此时没有准备好数据，它不会阻塞用户进程，而是立刻返回EWOULDBLOCK错误，用户进程收到这个error后又不断发起recvfrom调用，知道kernel中有数据准备好，此时内核将数据从内核拷贝到内存中，拷贝完成后返回OK给用户。所以，non-blocking I/O在数据准备阶段非阻塞，在拷贝阶段阻塞。
在non-blocking I/O模型中，用户进程轮询(polling)内核会耗费大量CPU时间。


### I/O multiplexing(I/O多路复用)
I/O多路复用一般用于处理多个连接的场景。在Linux系统中，有select, poll和epoll来具体实现多路复用。具体来说，select/poll/epoll轮询每个连接的文件描述符(file descriptor, fd),当某个连接上有数据到达，就立刻返回给用户进程，读操作流程如下：
![](http://images.cnitblog.com/i/434101/201406/281850258517851.jpg)
在图中，当用户进程调用select时，用户进程被阻塞，kernel轮询各个fd，当某个连接上有数据到达时，select返回。此时用户再调用recvfrom，数据从内核拷贝到内存。I/O操作在两个阶段都是阻塞的，只不过在等待数据到达时被select阻塞，在拷贝数据时被recvfrom阻塞。
所以，多路复用其实和阻塞I/O比较类似，而且，还更差一些， 因为多路复用中有两个system call(select和recvfrom)，而阻塞I/O中只有一个system call(recvfrom)。但是，多路复用的优点在于能处理更多的网络连接。在多路复用模型中，socket一般被设置为non-blocking。


### signal-driven I/O(信号驱动式I/O)
信号驱动式I/O的读操作流程如下：
![](http://images.cnitblog.com/i/434101/201406/281852082113755.jpg)
与之前的模型不同的是，信号驱动式I/O模型中，用户进程首先要通过sigaction系统调用安装信号处理函数，此时内核立即返回，用户进程不会被阻塞。当有数据到达并准备好时，内核发送一个SIGIO信号，通知用户进程处理数据。
信号驱动式I/O的优点是用户进程在等待数据到达的过程中不会被阻塞。当信号处理函数得到SIGIO通知后，用户进程再开始调用recvfrom读取数据。

### asynchronous I/O(异步I/O)
Linux下异步I/O实际上用得很少，读操作的流程如下：
![](http://images.cnitblog.com/i/434101/201406/281900431648409.jpg)
图中，当用户application发起aio_read调用后，系统调用立刻返回，用户经常不会被block。此后用户进程继续执行其它任务(process continues executing)，内核完成I/O操作的两步，等待数据到来，有数据准备好后将数据从内核拷贝到内存，并发送一个信号给用户，用户调用信号处理函数处理收到的数据。
在异步I/O中，用户发起读操作后就开始处理别的任务，直到收到内核发来的信号才处理读到的数据，整个过程没有发生阻塞。但是收到信号后的处理这一步可能会有一定的切换开销（这一点尚未得到证实）。


## （二）几种I/O模型的区别
首先来看blocking和non-blocking。blocking I/O在数据准备和拷贝数据时，用户进程都是阻塞的，而non-blocking I/O在数据准备时不会阻塞。
然后是同步和异步模型。POSIX对于同步和异步的定义如下：
> A synchronous I/O operation causes the requesting process to be blocked until that I/O operation completes.


> An asynchronous I/O operation does not cause the requesting process to be blocked.

同步和异步的区别在于，同步I/O的操作会阻塞调用线程，直到I/O操作完成，而异步I/O则完全不会阻塞调用线程。在这里值得注意的时，上面提到的non-blocking I/O实际上也是同步I/O，因为在数据从内核拷贝到内存时，recvfrom被阻塞了。

所以前面提到的blocking, non-blocking, I/O multiplexing和signal-driven的I/O模型都是同步I/O模型，只有asynchronous I/O是真正的异步I/O模型。这些I/O模型的比较如下：
![](http://hi.csdn.net/attachment/201007/31/0_1280551552NVgW.gif)

感谢经典的《UNIX网络编程》，以及IBM developerWorks。

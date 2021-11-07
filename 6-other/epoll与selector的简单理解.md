# epoll与selector的简单理解

**概念理解**

selector与epoll是多路复用的函数。我认为多路复用是针对bio而言，指的是通过单线程来追踪管理多个socket对象。传统的bio中，在socket的accept与read两个阶段都会造成阻塞，那么就无法处理并发问题，即仅一个socket对象就已经占用了IO对象，没有余力解决其他线程的请求。那么如何让bio能够处理并发问题呢？就是在accept和read阶段不再阻塞，当accept到socket对象的时候就将其缓存至list中，同时如果read到数据了就处理数据。但如果没有accept对象，则会去list中询问以前accept的对象有没有需要read的数据。如此，通过一个线程完成了多个socket对象的管理。那么selector与epoll就是完成了上述功能。

**epoll和select的区别**

进程通过将一个或多个fd传递给select或poll系统调用，阻塞在select;这样select/poll可以帮我们侦测许多fd是否就绪；但是select/poll是顺序扫描fd是否就绪，而且支持的fd数量有限。linux还提供了一个epoll系统调用，epoll是基于事件驱动方式，而不是顺序扫描,当有fd就绪时，立即回调函数rollback



如果没有I/O事件产生，我们的程序就会阻塞在select处。但是依然有个问题，我们从select那里仅仅知道了，有I/O事件发生了，但却并不知道是那几个流（可能有一个，多个，甚至全部），我们只能无差别轮询所有流，找出能读出数据，或者写入数据的流，对他们进行操作。

但是使用select，我们有O(n)的无差别轮询复杂度，同时处理的流越多，没一次无差别轮询时间就越长。再次

说了这么多，终于能好好解释epoll了

epoll可以理解为event poll，不同于忙轮询和无差别轮询，epoll之会把哪个流发生了怎样的I/O事件通知我们。此时我们对这些流的操作都是有意义的。（复杂度降低到了O(1)）

**对比分析**

selector内部维持了一个数据结构（bitmap），用来存储已经接受的socket对象。然后将此数据复制一份，交给内核去轮训每个socket对象是否有期待的事件发生，如果有就会进行置位（用户态到内核态的复制）。然后返回给用户态，由用户态完成事件的处理。

 

epoll针对selector进行了优化，采用红黑树来存储accept的socket对象，同时还维持着一个list，存放有发生的事件的socket以及事件，然后将此任务队列返回。用户态无需遍历所有的socket对象。下图即为epoll的说明图。

![img](https://img2020.cnblogs.com/i-beta/1506026/202003/1506026-20200308122904954-1443116064.png)
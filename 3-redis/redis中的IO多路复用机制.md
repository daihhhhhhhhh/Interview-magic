# redis中的IO多路复用机制

## 引言

提起`Redis`，我们经常会说其底层是一个单线程模型，但这是不严谨的。**`Redis` 单线程指的是网络请求模块使用了一个线程，即一个线程处理所有网络请求，其他模块该使用多线程，仍会使用了多个线程**。既然是单线程模型，那么`CPU`不是`Redis`的瓶颈。`Redis`的瓶颈最有可能是**机器内存**或者**网络带宽**。

## Redis中的单线程模型

`Redis`基于`Reactor`模式开发了自己的网络事件处理器，称之为**文件事件处理器(`File Event Hanlder`)**。文件事件处理器由`Socket`、`IO`多路复用程序、文件事件分派器(`dispather`)，事件处理器(`handler`)四部分组成。关于`IO`多路复用的相关知识，这方面可以参考我之前的[一篇文章](https://www.cnblogs.com/reecelin/p/13537734.html)，这里就不多解释了。文件事件处理器的模型如下所示：

[![image-20200820212146572](https://gitee.com/workingonescape/typora_images/raw/master/image-20200820212146572.png)](https://gitee.com/workingonescape/typora_images/raw/master/image-20200820212146572.png)

`IO`多路复用程序会同时监听多个`socket`，当被监听的`socket`准备好执行`accept`、`read`、`write`、`close`等操作时，与这些操作相对应的文件事件就会产生。`IO`多路复用程序会把所有产生事件的`socket`压入一个队列中，然后有序地每次仅一个`socket`的方式传送给文件事件分派器，文件事件分派器接收到`socket`之后会根据`socket`产生的事件类型调用对应的事件处理器进行处理。

文件事件处理器分为几种：

- **连接应答处理器**：用于处理客户端的连接请求；
- **命令请求处理器**：用于执行客户端传递过来的命令，比如常见的`set`、`lpush`等；
- **命令回复处理器**：用于返回客户端命令的执行结果，比如`set`、`get`等命令的结果；

事件种类：

- `AE_READABLE`

  ：与两个事件处理器结合使用。

  - 当客户端连接服务器端时，服务器端会将连接应答处理器与`socket`的`AE_READABLE`事件关联起来；
  - 当客户端向服务端发送命令的时候，服务器端将命令请求处理器与`AE_READABLE`事件关联起来；

- `AE_WRITABLE`：当服务端有数据需要回传给客户端时，服务端将命令回复处理器与`socket`的`AE_WRITABLE`事件关联起来。

`Redis`的客户端与服务端的交互过程如下所示：

[![image-20210917160156402](https://gitee.com/workingonescape/typora_images/raw/master/image-20210917160156402.png)](https://gitee.com/workingonescape/typora_images/raw/master/image-20210917160156402.png)


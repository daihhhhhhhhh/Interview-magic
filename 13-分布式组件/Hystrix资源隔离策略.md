# Hystrix资源隔离策略

在一个基于微服务的应用程序中，您通常需要调用多个微服务完成一个特定任务。不使用舱壁模式，这些调用默认是使用相同的线程来执行调用的，这些线程Java容器为处理所有请求预留的。在高服务器请求的情况下，一个性能较低的服务会“霸占”java容器中绝大多数线程，而其它性能正常的服务的请求则需要等待线程资源的释放。最后，整个java容器会崩溃。舱壁模式能将远程调用隔离在各个远程调用自己的线程池中，因此单个性能出问题的服务能得到控制，java容器也不会崩溃。

Hystrix将远程服务的请求托管在一个线程池中。即默认情况下，所有Hystrix命令(@HystrixCommand)共享同一个线程池来处理这些请求。该线程池中持有10个线程来处理各种远程服务请求，可以是REST服务调用、数据库访问等。如下图所示：

![img](https://img2018.cnblogs.com/blog/285763/201809/285763-20180920160336067-1551717122.jpg)

@HystrixCommand的默认配置适用于只有少量远程调用的应用。幸运的是，Hystrix提供了简单易用的方法实现舱壁来隔离不同的远程资源调用。下图说明了Hystrix将不同的远程调用隔离在不同的“舱室”（线程池）中：![img](https://img2018.cnblogs.com/blog/285763/201809/285763-20180920160353139-1937243496.jpg)

实现这种隔离的线程池，需要使用到@HystrixCommand注解提供的其他属性。接下来，我们会：

1. 为方法getLicensesByOrg()设置一个隔离的线程池
2. 设置该线程池的线程数
3. 设置队列的容量，该队列的作用是当线程池中的线程都处于工作状态，接下来的请求会进入该队列。
   下面的代码演示了如何自定义舱壁：



```
@HystrixCommand(
        fallbackMethod = "buildFallbackLicenseList",
        threadPoolKey = "licenseByOrgThreadPool",
        threadPoolProperties = {
            @HystrixProperty(name = "coreSize",value="30"),
            @HystrixProperty(name="maxQueueSize", value="10")
        }
)
public List<License> getLicensesByOrg(String organizationId){
    randomlyRunLong();
    return licenseRepository.findByOrganizationId(organizationId);
}
```



 

资源隔离的两种策略

## **1.依赖隔离概述**

依赖隔离是Hystrix的核心目的。依赖隔离其实就是资源隔离，把对依赖使用的资源隔离起来，统一控制和调度。那为什么需要把资源隔离起来呢？主要有以下几点：

1.合理分配资源，把给资源分配的控制权交给用户，某一个依赖的故障不会影响到其他的依赖调用，访问资源也不受影响。

2.可以方便的指定调用策略，比如超时异常，熔断处理。

3.对依赖限制资源也是对下游依赖起到一个保护作用，避免大量的并发请求在依赖服务有问题的时候造成依赖服务瘫痪或者更糟的雪崩效应。

4.对依赖调用进行封装有利于对调用的监控和分析，类似于hystrix-dashboard的使用。

 

Hystrix提供了两种依赖隔离方式：线程池隔离 和 信号量隔离。两种隔离方式都是限制对共享资源的并发访问量，线程在就绪状态、运行状态、阻塞状态、终止状态间转变时需要由操作系统调度，占用很大的性能消耗；而信号量是在访问共享资源时，进行tryAcquire，tryAcquire成功才允许访问共享资源。

如下图，线程池隔离，Hystrix可以为每一个依赖建立一个线程池，使之和其他依赖的使用资源隔离，同时限制他们的并发访问和阻塞扩张。每个依赖可以根据权重分配资源（这里主要是线程），每一部分的依赖出现了问题，也不会影响其他依赖的使用资源。

![img](https://img2018.cnblogs.com/blog/285763/201809/285763-20180920152729826-512299129.png)

## **2.线程池隔离**

如果简单的使用异步线程来实现依赖调用会有如下问题：1、线程的创建和销毁；2、线程上下文空间的切换，用户态和内核态的切换带来的性能损耗。

使用线程池的方式可以解决第一种问题，但是第二个问题计算开销是不能避免的。Netflix在使用过程中详细评估了使用异步线程和同步线程带来的性能差异，结果表明在99%的情况下，异步线程带来的几毫秒延迟的完全可以接受的。

![img](https://img2018.cnblogs.com/blog/285763/201809/285763-20180920152756556-1433968953.png)

## **3.线程池隔离的优缺点**

优点：

- 一个依赖可以给予一个线程池，这个依赖的异常不会影响其他的依赖。
- 使用线程可以完全隔离第三方代码,请求线程可以快速放回。
- 当一个失败的依赖再次变成可用时，线程池将清理，并立即恢复可用，而不是一个长时间的恢复。
- 可以完全模拟异步调用，方便异步编程。
- 使用线程池，可以有效的进行实时监控、统计和封装。

缺点：

- 使用线程池的缺点主要是增加了计算的开销。每一个依赖调用都会涉及到队列，调度，上下文切换，而这些操作都有可能在不同的线程中执行。

## **4.Command Name&Command Group**

Hystrix使用Command模式对依赖调用进行封装。当我们写一个调用继承HystrixCommand的时候，可以指定一个名称Command Name。如果不指定Hystrix将会使用getClass().getSimpleName()来默认获取。如果要指定，可以使用如下代码，使用HystrixCommandKey.Factory帮助类在构造函数中指定。



```
public HelloWorldCommand(String name)
    {
        //定义命令组 和 方法调用超时时间
        super(Setter.withGroupKey(HystrixCommandGroupKey.Factory.asKey("HelloWorldGroup"))
                .andCommandKey(HystrixCommandKey.Factory.asKey("HelloWorldCommand")));
        this.name = name;
    }
```



而Command Group可以把一组Command归为一组。如上例代码，可以使用HystrixCommandGroupKey.Factory.asKey来指定Command Group。一般情况下，逻辑上是同一类型的会放在同一个Command Group中。比如，获取用户相关信息的依赖可以放在一起。

虽然Hystrix可以为每个依赖建立一个线程池，但是如果依赖成千上万，建立那么多线程池肯定是不可能的。所以默认情况下，Hystrix会为每一个Command Group建立一个线程池。

 

## **5.Command Thread Pool**

Hystrix可以指定创建或关联上一个线程池，每一个线程池都有一个Key。这个线程池就是线程隔离的关键，所有的监控、缓存、调用等等都来自于这个线程池。可以通过如下代码指定线程池：



```
public HelloWorldCommand(String name)
    {
        //定义命令组 和 方法调用超时时间
        super(Setter.withGroupKey(HystrixCommandGroupKey.Factory.asKey("HelloWorldGroup"))
                .andCommandKey(HystrixCommandKey.Factory.asKey("HelloWorldCommand"))
                .andThreadPoolKey(HystrixThreadPoolKey.Factory.asKey("HelloWorldPool")));
        this.name = name;
    }
```



上文说到，默认情况下，每一个Command Group会自动创建一个线程池。那什么时候我们需要单独指定线程池呢？因为线程池主要的目的是隔离，所以当有一些依赖在一个Command Group中，但是又有隔离的必要的时候，比如一个依赖的超时会用满所有的线程池线程，而不应该影响其他的依赖。

 

## **6.基本实现原理**

Command模式：Hystrix中大量使用rxjava来实现Command模式。所有自定义的Command，不管继承于HystrixObservableCommand还是HystrixCommand,最终都继承于AbstractCommand。Thread Pool，Command Group，Command Key都在AbstractCommand这里实现。

 

线程池的创建和管理：Hystrix的线程池在HystrixConcurrencyStrategy初始化，线程池是由ThreadPoolExecutor实现的。每个线程池默认初始化10个线程。Hystrix有个静态类Factory，创建的线程池会被存储在Factory中的ConcurrentHashMap中。ConcurrentHashMap的Key则是上文说到的CommandGroupKey或者指定的ThreadPoolKey。每次命令执行的时候，都会根据ThreadPoolKey去找到对应的线程池。线程池拥有一个继承于rxjava中Scheduler的HystrixContextScheduler，用于在执行命令的时候，把命令在这个线程池上调度执行。

 

命令的执行：以execute()方法为例，Hystrix通过toObservable()来构造命令，构造过程中，定义了整个命令执行过程中的stage（未开始、执行中、完成执行、执行异常等等）的回调和处理方法。在executeCommandWithSpecifiedIsolation()方法中，使用exjava的subscribeOn方法，传入上文提到的HystrixContextScheduler对象，通过HystrixContextScheduler的ThreadPoolScheduler把命令submit到ThreadPoolExecutor中去执行。

 

## **7.最佳实践**

对于那些本来延迟就比较小的请求（例如访问本地缓存成功率很高的请求）来说，线程池带来的开销是非常高的，这时，你可以考虑采用其他方法，例如非阻塞信号量（不支持超时），来实现依赖服务的隔离，使用信号量的开销很小。但绝大多数情况下，Netflix 更偏向于使用线程池来隔离依赖服务，因为其带来的额外开销可以接受，并且能支持包括超时在内的所有功能。

 

在一个分布式系统中，服务之间都是相互调用的，如下图所示：

![img](https://img2018.cnblogs.com/blog/285763/201809/285763-20180920153036793-1428721224.png)

例如，我们容器(Tomcat)配置的线程个数为1000，服务A-服务R，其中服务I的并发量非常的大，需要500个线程来执行，此时，服务I又挂了，那么这500个线程很可能就夯死了，那么剩下的服务，总共可用的线程为500个，随着并发量的增大，剩余服务挂掉的风险就会越来越大，最后导致整个系统的所有服务都不可用，直到系统宕机。这就是服务的雪崩效应。Hystrix就是用来做资源隔离的，比如说，当客户端向服务端发送请求时，给服务I分配了10个线程，只要超过了这个并发量就走降级服务，就算服务I挂了，最多也就导致服务I不可用，容器的10个线程不可用了，但是不会影响系统中的其他服务。下面，我们就来具体说下这两种隔离策略：

1、线程池

线程池隔离的示意图如下：

![img](https://img2018.cnblogs.com/blog/285763/201809/285763-20180920153051359-1383492913.png)

上图的左边2/3是线程池资源隔离示意图，右边的1/3是信号量资源隔离示意图，我们先来看左边的示意图。

当用户请求服务A和服务I的时候，tomcat的线程(图中蓝色箭头标注)会将请求的任务交给服务A和服务I的内部线程池里面的线程(图中橘色箭头标注)来执行，tomcat的线程就可以去干别的事情去了，当服务A和服务I自己线程池里面的线程执行完任务之后，就会将调用的结果返回给tomcat的线程，从而实现资源的隔离，当有大量并发的时候，服务内部的线程池的数量就决定了整个服务的并发度，例如服务A的线程池大小为10个，当同时有12请求时，只会允许10个任务在执行，其他的任务被放在线程池队列中，或者是直接走降级服务，此时，如果服务A挂了，就不会造成大量的tomcat线程被服务A拖死，服务I依然能够提供服务。整个系统不会受太大的影响。

2、信号量

 execution.isolation.strategy: "SEMAPHORE"
 hystrix.command.default.execution.isolation.thread.timeoutInMilliseconds: 3000
 execution.isolation.semaphore.maxConcurrentRequests: 100

信号量的资源隔离只是起到一个开关的作用，例如，服务X的信号量大小为10，那么同时只允许10个tomcat的线程(此处是tomcat的线程，而不是服务X的独立线程池里面的线程)来访问服务X，其他的请求就会被拒绝，从而达到限流保护的作用。

3、二者的比较

 

|              | 线程池隔离                                                   | 信号量隔离                                                   |
| ------------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
| 线程         | 与调用线程非相同线程                                         | 与调用线程相同（jetty线程）                                  |
| 开销         | 排队、调度、上下文开销等                                     | 无线程切换，开销低                                           |
| 异步         | 可以是异步，也可以是同步。看调用的方法                       | 同步调用，不支持异步                                         |
| 并发支持     | 支持（最大线程池大小hystrix.threadpool.default.maximumSize） | 支持（最大信号量上限maxConcurrentRequests）                  |
| 是否超时     | 支持，可直接返回                                             | 不支持，如果阻塞，只能通过调用协议（如：socket超时才能返回） |
| 是否支持熔断 | 支持，当线程池到达maxSize后，再请求会触发fallback接口进行熔断 | 支持，当信号量达到maxConcurrentRequests后。再请求会触发fallback |
| 隔离原理     | 每个服务单独用线程池                                         | 通过信号量的计数器                                           |
| 资源开销     | 大，大量线程的上下文切换，容易造成机器负载高                 | 小，只是个计数器                                             |

 

4、总结

当请求的服务网络开销比较大的时候，或者是请求比较耗时的时候，我们最好是使用线程隔离策略，这样的话，可以保证大量的容器(tomcat)线程可用，不会由于服务原因，一直处于阻塞或等待状态，快速失败返回。而当我们请求缓存这些服务的时候，我们可以使用信号量隔离策略，因为这类服务的返回通常会非常的快，不会占用容器线程太长时间，而且也减少了线程切换的一些开销，提高了缓存服务的效率。

# 异步RPC

异步RPC主要目的是提高并发，比如你的接口，内部调用了3个服务，时间分别为T1, T2, T3。如果是顺序调用，则总时间是T1 + T2 + T3；如果并发调用，总时间是Max(T1,T2,T3)。

当然，这里有1个前提条件，这3个调用直接，互相不依赖。

同样，一般成熟的RPC框架，本身都提高了异步化接口，Future或者Callback形式。

同样，Hystrix也提高了同步调用、异步调用方式，
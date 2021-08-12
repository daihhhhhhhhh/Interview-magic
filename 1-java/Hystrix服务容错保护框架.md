# Hystrix服务容错保护框架

在微服务架构中，我们将系统拆分成了一个个的服务单元，各单元应用间通过服务注册与订阅的方式互相依赖。由于每个单元都在不同的进程中运行，依赖通过远程调用的方式执行，这样就有可能因为网络原因或是依赖服务自身问题出现调用故障或延迟，而这些问题会直接导致调用方的对外服务也出现延迟，若此时调用方的请求不断增加，最后就会出现因等待出现故障的依赖方响应而形成任务积压，线程资源无法释放，最终导致自身服务的瘫痪，进一步甚至出现故障的蔓延最终导致整个系统的瘫痪。如果这样的架构存在如此严重的隐患，那么相较传统架构就更加的不稳定。为了解决这样的问题，因此产生了断路器等一系列的服务保护机制。

针对上述问题，在Spring Cloud Hystrix中实现了线程隔离、断路器等一系列的服务保护功能。它也是基于Netflix的开源框架 Hystrix实现的，该框架目标在于通过控制那些访问远程系统、服务和第三方库的节点，从而对延迟和故障提供更强大的容错能力。Hystrix具备了服务降级、服务熔断、线程隔离、请求缓存、请求合并以及服务监控等强大功能。

接下来，我们就从一个简单示例开始对Spring Cloud Hystrix的学习与使用。

# Hystrix是什么

保护框架
 在分布式中能实现对服务容错保护。
 减少服务之间的依赖关系,(不是业务上的依赖)防止雪崩效应,最终以服务降级,熔断,限流处理.

# 容错概念:

在服务发生不可用的时候,给出合理的处理方案:预备方案.



![img](https:////upload-images.jianshu.io/upload_images/4055666-1b8c6f69989f23c6.png?imageMogr2/auto-orient/strip|imageView2/2/w/544/format/webp)

image.png

# 服务雪崩效应

当一个服务突然收到高并发请,tomcat服务器如果承受不了的情况下产生服务堆积.可能导致其他服务不可用.

# 截图

![img](https:////upload-images.jianshu.io/upload_images/4055666-d884362ec81a20c7.png?imageMogr2/auto-orient/strip|imageView2/2/w/913/format/webp)

image.png

# 截图

![img](https:////upload-images.jianshu.io/upload_images/4055666-487c41a356d192c0.png?imageMogr2/auto-orient/strip|imageView2/2/w/538/format/webp)

image.png

# 服务保护

![img](https:////upload-images.jianshu.io/upload_images/4055666-4232a8724bd00f05.png?imageMogr2/auto-orient/strip|imageView2/2/w/1044/format/webp)

image.png

# 服务隔离

![img](https:////upload-images.jianshu.io/upload_images/4055666-188295ba7a59a67d.png?imageMogr2/auto-orient/strip|imageView2/2/w/972/format/webp)

image.png

# 雪崩效应:

多个请求到同一线程池中，导致服务堆积，使接下来的服务请求没有线程继续处理，所以产生雪崩效应。

# 解决雪崩效应（服务隔离）

每个接口互不影响，

- 线程池方式：每个服务都有自己独立的线程池来管理运行自己的接口。

  ![img](https:////upload-images.jianshu.io/upload_images/4055666-b5828730eaaa3d2a.png?imageMogr2/auto-orient/strip|imageView2/2/w/812/format/webp)

  image.png

- 计数器方式：使用原子计数器，这对于每个服务设置一个阈值，当服务请求达到这个上线时直接返回一个错误提示。

  ![img](https:////upload-images.jianshu.io/upload_images/4055666-3a34d1fb673d02ec.png?imageMogr2/auto-orient/strip|imageView2/2/w/550/format/webp)

  image.png

  ![img](https:////upload-images.jianshu.io/upload_images/4055666-dc5d061971f99f2d.png?imageMogr2/auto-orient/strip|imageView2/2/w/1078/format/webp)

  image.png

服务的个例有两种实现方式，线程池方式 计数器方式
 不同的服务之间，服务的调用不产生影响。
 线程池方式：
 每个接口服务之间都有一个线程池来管理运行自己的接口。确点时菜谱的开销比较大，但是可以解决雪崩效应。

计数器方式：
 使用原子计数器，会计算当前的所有线程总数，设置一个上限值，当增加线程的时候计数器+1，当线程运行完后减计数器-1，当超过请求队列的上限值以后会调用自己的拒绝策略。

# 服务降级

当前服务不可用（），导致客户端进入等待状态，服务降级就是为了不让客户端进行等待，从而返回一个错误提示给客户端。

- 作用：
   目的是提高用户体验（示例错误提示：服务器忙请稍后再试）。

![img](https:////upload-images.jianshu.io/upload_images/4055666-0465e392535054be.png?imageMogr2/auto-orient/strip|imageView2/2/w/790/format/webp)

image.png

# 防止服务雪崩效应利器Hystrix

# 引入依赖hystrix jar包

![img](https:////upload-images.jianshu.io/upload_images/4055666-b7f1d013f9e6ee7e.png?imageMogr2/auto-orient/strip|imageView2/2/w/513/format/webp)

image.png

# 编写服务端class需要继承 hystrixCommand类

![img](https:////upload-images.jianshu.io/upload_images/4055666-e1cbf58ba7a6901d.png?imageMogr2/auto-orient/strip|imageView2/2/w/695/format/webp)

image.png

![img](https:////upload-images.jianshu.io/upload_images/4055666-cbbbe38289a5d6ff.png?imageMogr2/auto-orient/strip|imageView2/2/w/1031/format/webp)

image.png
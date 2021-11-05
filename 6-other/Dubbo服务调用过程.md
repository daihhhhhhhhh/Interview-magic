# Dubbo服务调用过程

节点角色说明：

Provider: 暴露服务的服务提供方。
Consumer: 调用远程服务的服务消费方。
Registry: 服务注册与发现的注册中心。
Monitor: 统计服务的调用次调和调用时间的监控中心。
Container: 服务运行容器。
调用关系说明：

0. 服务容器负责启动，加载，运行服务提供者。
1. 服务提供者在启动时，向注册中心注册自己提供的服务。
2. 服务消费者在启动时，向注册中心订阅自己所需的服务。
3. 注册中心返回服务提供者地址列表给消费者，如果有变更，注册中心将基于长连接推送变更数据给消费者。
4. 服务消费者，从提供者地址列表中，基于软负载均衡算法，选一台提供者进行调用，如果调用失败，再选另一台调用。
5. 服务消费者和提供者，在内存中累计调用次数和调用时间，定时每分钟发送一次统计数据到监控中心。
基本上已经把整个过程说得很清楚了。

在此基础上，使用ZooKeeper作为注册中心，通过之前的Demo实例（Spring Boot2整合Dubbo）演示整个过程。

1、首先，启动服务提供者模块dubbo-provider。这时候，向注册中心注册自己提供的服务，也就是，在ZooKeeper上创建对应的临时子节点。

打开ZooKeeper可视化客户端，可以看到这样子一个目录结构：

![](https://img-blog.csdnimg.cn/201901052133541.png)

在dubbo根节点下，会产生一个以接口名称命名的子节点，该节点为永久节点（因为接口名称不会经常变化）。接口节点下面，会有四个子节点，分别表示消费者、配置项、路由器、提供者。提供者节点下面，会有一个dubbo协议地址的子节点，这个节点就是我们启动服务容器时注册的服务了，如果还有其它的该接口实现类服务提供者，就会在同级目录再创建一个临时节点。为什么是临时节点呢？第一，服务提供者的地址是经常发生变化的；第二，如果提供者的服务器宕机了，跟ZooKeeper会话结束导致临时节点被删除，就可以自动通知注册了该节点事件监听的客户端。

附上Dubbo-Admin上面的信息

![](https://img-blog.csdnimg.cn/20190105220507851.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzMxMTQyNTUz,size_16,color_FFFFFF,t_70)

2、然后，启动服务消费者模块dubbo-consumer，这时候，向注册中心订阅自己所需的服务，也就是，获取ZooKeeper的对应目录子节点，然后保存到本地。

再看ZooKeeper的目录结构：

![](https://img-blog.csdnimg.cn/20190105220651807.png)

在consumers节点下多了一个临时节点（采用临时节点是因为客户端断开连接时可以自动删除） 。

![](https://img-blog.csdnimg.cn/20190105220741490.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzMxMTQyNTUz,size_16,color_FFFFFF,t_70)

3、服务调用的时候，在本地的服务提供列表中，基于软负载（RPC框架基本上都提供了本地负载均衡功能） 算法，找到合适的服务，然后进行远程调用。如果调用失败，选择另一台服务进行调用，这是容错功能。

Dubbo提供了多种负载均衡算法和容错采用措施提供配置。

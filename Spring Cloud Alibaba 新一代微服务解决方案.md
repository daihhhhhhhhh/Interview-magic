# Spring Cloud Alibaba 新一代微服务解决方案

## 1、Spring Cloud Alibaba 是什么

Spring Cloud Alibaba 是阿里巴巴提供的微服务开发一站式解决方案，是阿里巴巴开源中间件与 Spring Cloud 体系的融合。

马老师左手双十一，右手阿里开源组件，不仅占据了程序员的购物车，还要攻占大家的开发工具。

### 先说说 Spring Cloud

提起微服务，不得不提 Spring Cloud 全家桶系列，SpringCloud 是若干个框架的集合，包括 spring-cloud-config、spring-cloud-bus 等近 20 个子项目，提供了服务治理、服务网关、智能路由、负载均衡、断路器、监控跟踪、分布式消息队列、配置管理等领域的解决方案。

Spring Cloud 通过 Spring Boot 风格的封装，屏蔽掉了复杂的配置和实现原理，最终给开发者留出了一套简单易懂、容易部署的分布式系统开发工具包。

一般来说，Spring Cloud 包含以下组件，主要以 Netflix 开源为主：



![img](https://pic4.zhimg.com/80/v2-2d78afcc4e7019b9788478345059a9a3_720w.jpg)



### Spring Cloud Alibaba

同 Spring Cloud 一样，Spring Cloud Alibaba 也是一套微服务解决方案，包含开发分布式应用微服务的必需组件，方便开发者通过 Spring Cloud 编程模型轻松使用这些组件来开发分布式应用服务。

依托 Spring Cloud Alibaba，您只需要添加一些注解和少量配置，就可以将 Spring Cloud 应用接入阿里微服务解决方案，通过阿里中间件来迅速搭建分布式应用系统。

作为 Spring Cloud 体系下的新实现，Spring Cloud Alibaba 跟官方的组件或其它的第三方实现如 Netflix, Consul，Zookeeper 等对比，具备了更多的功能:



![img](https://pic4.zhimg.com/80/v2-3a132c503bb7cde19314acc8a4866c4b_720w.jpg)



## 2、Spring Cloud Alibaba 包含组件

这幅图是 Spring Cloud Alibaba 系列组件，其中包含了阿里开源组件，阿里云商业化组件，以及集成Spring Cloud 组件。



![img](https://pic4.zhimg.com/80/v2-46c0b9e0d41c441d222390c79a4cd53b_720w.jpg)



### 阿里开源组件

Nacos：一个更易于构建云原生应用的动态服务发现、配置管理和服务管理平台。

Sentinel：把流量作为切入点，从流量控制、熔断降级、系统负载保护等多个维度保护服务的稳定性。

RocketMQ：开源的分布式消息系统，基于高可用分布式集群技术，提供低延时的、高可靠的消息发布与订阅服务。

Dubbo：这个就不用多说了，在国内应用非常广泛的一款高性能 Java RPC 框架。

Seata：阿里巴巴开源产品，一个易于使用的高性能微服务分布式事务解决方案。

Arthas：开源的Java动态追踪工具，基于字节码增强技术，功能非常强大。

### 阿里商业化组件

作为一家商业公司，阿里巴巴推出 Spring Cloud Alibaba，很大程度上市希望通过抢占开发者生态，来帮助推广自家的云产品。所以在开源社区，夹带了不少私货，这部分组件我在阿里工作时都曾经使用过，整体易用性和稳定性还是很高的。

Alibaba Cloud ACM：一款在分布式架构环境中对应用配置进行集中管理和推送的应用配置中心产品。

Alibaba Cloud OSS：阿里云对象存储服务（Object Storage Service，简称 OSS），是阿里云提供的云存储服务。

Alibaba Cloud SchedulerX：阿里中间件团队开发的一款分布式任务调度产品，提供秒级、精准的定时（基于 Cron 表达式）任务调度服务。

### 集成 Spring Cloud 组件

Spring Cloud Alibaba 作为整套的微服务解决组件，只依靠目前阿里的开源组件是不够的，更多的是集成当前的社区组件，所以 Spring Cloud Alibaba 可以集成 Zuul，OpenFeign等网关，也支持 Spring Cloud Stream 消息组件。

## 3、Spring Cloud Alibaba 功能

那么作为微服务解决方案， Spring Cloud Alibaba是如何支持微服务治理的各个功能。

### 服务注册与发现

Spring Cloud Alibaba 基于 Nacos 提供 spring-cloud-alibaba-starter-nacos-discovery & spring-cloud-alibaba-starter-nacos-config 实现了服务注册 & 配置管理功能。依靠 @EnableDiscoveryClient 进行服务的注册，兼容 RestTemplate & OpenFeign 的客户端进行服务调用。

适配 Spring Cloud 服务注册与发现标准，默认集成了 Ribbon 的支持。

### 支持多协议的服务调用

Spring Cloud 默认的服务调用依赖 OpenFeign 或 RestTemplate 使用 REST 进行调用。

使用 @DubboTransported 注解可将底层的 Rest 协议无缝切换成 Dubbo RPC 协议，进行 RPC 调用。

```text
@FeignClient("dubbo-provider")
@DubboTransported(protocol = "dubbo")
public interface DubboFeignRestService {
  @GetMapping(value = "/param")
  String param(@RequestParam("param") String param);

  @PostMapping("/saveB")
  String saveB(@RequestParam("a") int a, @RequestParam("b") String b);
}
```

### 服务限流降级

作为稳定性的核心要素之一，服务限流和降级是微服务领域特别重要的一环，Spring Cloud Alibaba 基于 Sentinel，对 Spring 体系内基本所有的客户端，网关进行了适配，

默认支持 WebServlet、WebFlux, OpenFeign、RestTemplate、Spring Cloud Gateway, Zuul, Dubbo 和 RocketMQ 限流降级功能的接入。

Sentinel应用比较简单，只需引入 starter，即可生效，可以在运行时通过控制台实时修改限流降级规则，还支持查看限流降级 Metrics 监控。

### 微服务消息驱动

支持为微服务应用构建消息驱动能力，基于 Spring Cloud Stream 提供 Binder 的新实现: Spring Cloud Stream RocketMQ Binder，

也新增了 Spring Cloud Bus 消息总线的新实现 Spring Cloud Bus RocketMQ。

### 分布式事务

使用 Seata 解决微服务场景下面临的分布式事务问题。

使用 @GlobalTransactional 注解，在微服务中传递事务上下文，可以对业务零侵入地解决分布式事务问题。

### 阿里云提供的商业能力

通过上面提到的OSS，schedulerx等组件，开发者可以在阿里云上实现对象存储，分布式任务调度等功能。

## 4、为什么我看好 Spring Cloud Alibaba

Spring Cloud Alibaba 虽然诞生时间不久，但是背靠大树好乘凉，赖于阿里巴巴强大的技术影响力，已经成为微服务解决方案的重要选择之一。

我认为 Spring Cloud Alibaba 的优势有以下几点：

### 阿里巴巴强大的技术输出能力

阿里巴巴无疑是国内开源技术领域的最有影响力的公司之一，已经有Dubbo、Druid，FastJson等成功的开源组件，
再加上阿里不遗余力的推广，社区发展也非常快。

### 集成Dubbo，利用Dubbo在微服务领域的超高人气

Dubbo是国内应用最广的分布式服务框架之一，基于Dubbo改造的Dubbox等也有很多公司在使用，

Spring Cloud Alibaba对Dubbo做了比较好的集成，可以吸引不少使用Dubbo的开发者。

### 云原生趋势，集成阿里云商业化组件

云原生（Cloud Native）是今年技术领域特别热门的一个词，云原生是一种专门针对云上应用而设计的方法，用于构建和部署应用，以充分发挥云计算的优势。

Spring Cloud Alibaba 集成了阿里云的商业化组件，可以说天然支持云原生特性。


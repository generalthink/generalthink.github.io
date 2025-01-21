---
title: 带你走进Spring Cloud的世界
date: 2025-01-21 11:40:42
tags: [Spring Cloud, Alibaba Spring Cloud]
---

开局我们先看一张Spring 社区发布的一张简化的架构图

![SpringCloudJ架构图](/images/spring-cloud/springcloud-architecture.jpg)

可以看到Spring Boot 通过自动装配和各种开箱即用的特性，搞定了数据层访问、RESTful 接口、日志组件、内
置容器等等基础功能，让开发人员不费吹灰之力就可以搭建起一个应用；Spring Cloud 主外，在应用集群之外提供了各种分布式系统的支持特性，帮助轻松实现负载均衡、熔断降级、配置管理等诸多微服务领域的功能。

从两者分工可以看出来， Spring Cloud就是在Spring Boot基础上添加了支持微服务领域的功能，从而变成了微服务领域的全家桶解决方案。

### Spring Cloud组件库的变更

Netflix 是一家美国的流媒体巨头，它靠着自己强大的技术实力，开发沉淀了一系列优秀的组件，这些组件经历了 Netflix 线上庞大业务规模的考验，功能特性和稳定性过硬。如 Eureka 服务注册中心、Ribbon 负载均衡器、Hystrix 服务容错组件等。后来发生的故事可能你已经猜到了，Netflix 将这些组件贡献给了 Spring 开源社区，构成了 Netflix 组件库。可以这么说，在 Spring Cloud 的早期阶段，是 Netflix 打下了的半壁江山。

后来Eureka 2.0 开源计划搁浅，而后 Netflix 宣布 Hystrix 进入维护状态，Eureka 和 Hystrix 这两款 Netflix 组
件库的明星项目停止了新功能的研发。

Spring 社区不得不开始思考替代方案，在后续的新版本中走向了“去 Netflix 化”。以至于 Netflix 的网关组件 Zuul 2.0 历经几次跳票千呼万唤始出来后，Spring Cloud 社区已经不打算集成 Zuul 2.0，而是掏出了自己的Gateway 网关。


Spring Cloud Alibaba 是由 Alibaba 贡献的组件库，随着阿里在开源路线上的持续投入，近几年阿里系在开源领域的声音非常响亮。Spring Cloud Alibaba 凝聚了阿里系在电商
领域超高并发经验的重量级组件，保持了旺盛的更新活力，成为了 Spring Cloud 社区的一股新生代力量，逐渐取代了旧王 Netflix 的江湖地位。


| **功能特性**    | **Alibaba组件** | **Netflix组件** | **SpringCloud官方或第三方开源组件库** |
|-------------|---------------|---------------|----------------------------|
| **服务治理**    | Nacos         | Eureka        | Consul                     |
| **负载均衡**    |               | Ribon         | LoadBalancer               |
| **服务间调用**   | Dubbo(RPC)    | Netflix Feign | OpenFeign                  |
| **服务容错**    | Sentinel      | Hystrix       | Resilence4j                |
| **配置中心**    | Nacos         |               | SpringCloud config         |
| **消息总线**    |               |               | Bus                        |
| **服务网关**    |               | Zuul          | Gateway                    |
| **分布式链路追踪** |               |               | Sleuth Zipkin              |
| **消息事件驱动**  | RockMQ        |               | Stream                     |
| **分布式事务**   | Seata         |               |                            |


上面表格中列出的业务开发过程中的常用功能性组件， 当然我们平时的使用中也有很多其他的开源组件可供选择，比如消息中间件还可以使用Kafaka。 


### Spring Cloud版本更新策略

我们来看看Spring Cloud各个版本号以及其更新节奏



| 版本代号 | 版本号 | 发布时间 |
|---------|-------|---------|
| Angel    | 1.0.x  | 2015年5月 |
| Brixton  | 1.1.x  | 2016年5月 |
| Camden   | 1.2.x  | 2017年7月 |
| Dalston  | 1.3.x  | 2017年4月 |
| Edgware  | 1.4.x  | 2017年11月 |
| Finchley | 2.0.x  | 2018年6月 |
| Greenwich| 2.1.x  | 2019年1月 |
| Hoxton   | 2.2.x  | 2019年11月 |
| Ilford   | 2020.0.x | 2020年12月 |
| Jubilee  | 2021.0.x | 2021年12月 |
| Kilburn  | 2022.0.x | 2022年11月 |
| Leyton   | 2023.0.x | 2023年11月 |
| Moorgate | 2024.0.x | 2024年12月 |
| Northfields | 2054.0.x | 预计2025 |

> 具体的可以参考 https://github.com/spring-cloud/spring-cloud-release/wiki/Spring-Cloud-Release-Train-Naming-Convention


#### 和springboot版本对应关系

| Spring Cloud 版本 | Spring Boot 版本 |
|-------------------|-------------------|
| 2025.0.x (Northfields) | 待定 |
| 2024.0.x (Moorgate) | 待定 |
| 2023.0.x (Leyton) | 3.2.x |
| 2022.0.x (Kilburn) | 3.0.x, 3.1.x |
| 2021.0.x (Jubilee) | 2.6.x, 2.7.x |
| 2020.0.x (Ilford) | 2.4.x, 2.5.x |
| Hoxton | 2.2.x, 2.3.x |
| Greenwich | 2.1.x |
| Finchley | 2.0.x |
| Edgware | 1.5.x |
| Dalston | 1.5.x |

> 你可以从https://spring.io/projects/spring-cloud看到他们之间的对应关系。

创建项目的时候你可以通过https://start.spring.io/，这样当选择 Spring Boot 版本时，它会自动显示兼容的 Spring Cloud 版本。 

或者通过https://spring.io/blog 查看一些信息。



#### Alibaba Spring Cloud版本对应关系

如果使用alibaba springcloud对应关系如下:

| Spring Cloud Alibaba Version | Spring Cloud Version | Spring Boot Version | JDK Version |
|------------------------------|----------------------|---------------------|-------------|
| 2023.0.0.0                   | Spring Cloud 2023.0.0| 3.2.x               | JDK 17+     |
| 2022.0.0.0                   | Spring Cloud 2022.0.0| 3.0.x               | JDK 17+     |
| 2021.0.5.0                   | Spring Cloud 2021.0.5| 2.6.13              | JDK 8+      |
| 2021.0.4.0                   | Spring Cloud 2021.0.4| 2.6.11              | JDK 8+      |
| 2021.0.1.0                   | Spring Cloud 2021.0.1| 2.6.3               | JDK 8+      |
| 2.2.7.RELEASE                | Hoxton.SR12          | 2.3.12.RELEASE      | JDK 8       |
| 2.2.6.RELEASE                | Hoxton.SR9           | 2.3.2.RELEASE       | JDK 8       |
| 2.1.4.RELEASE                | Greenwich.SR6        | 2.1.13.RELEASE      | JDK 8       |
| 2.0.4.RELEASE                | Finchley.SR4         | 2.0.9.RELEASE       | JDK 8       |


> 可以通过 https://github.com/alibaba/spring-cloud-alibaba 这里查看更详细的对应关系

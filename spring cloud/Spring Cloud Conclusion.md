# Spring Cloud Conclusion

&emsp;&emsp;`Spring Cloud` 为开发人员提供了快速构建分布式微服务系统的一些工具，包括配置管理、服务发现、断路器、路由、微代理、事件总线、全局锁、决策竞选、分布式会话等等。它运行环境简单，可以在开发人员的电脑上跑。另外`Spring Cloud`是基于`Spring Boot`的，所以需要开发中对`Spring Boot`有一定的了解。

**Spring Cloud 学习初步结论**：

* 用 etcd 或 Consul 来代替 Eureka/Zookeeper 作为微服务注册和配置中心
* 用 Ribbon(负载均衡) + Feign(快速实现) + Hystrix(断路器) 来实现微服务代码，是客户端实现的
* 用 Actuator、Sleuth 和 Zipkkin 来实现监控和调用链路跟踪
* 服务器
  * 一套 etcd 或 Consul 集群，用于服务注册和配置信息存储
  * 一台配置服务器，用于更改配置？
  * 一台 Hystrix Turbine 服务调用监控服务器
  * 一台 Zipkin 分布式调用链路分析服务器
  * 一套 RabbitMQ/Kafka 集群用于消息总线
  * 一套 Zuul 集群来作为前端 API Gateway 做鉴权和代理
  * 一堆用 Ribbon(负载均衡) + Feign(快速实现) + Hystrix(断路器) 实现的微服务


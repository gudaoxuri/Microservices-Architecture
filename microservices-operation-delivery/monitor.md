# 服务监控

服务监控是保证生产环境稳定最基础也是最重要的措施，微服务下尤是。如果没有成熟的一套监控方案那微服务化注定会失败！

服务监控的核心是监控什么？基础系统监控包含节点的硬盘/网络IO、CPU/内存/硬盘占用等，针对JVM的微服务我们还要特别关注以下指标：

* 类的loaded与unloaded
* 内存的used/committed/max
* 垃圾回收Minor/Full GC的频率、次数、暂停时间
* 线程的峰值数量、当前数量，守护线程的数量

这些信息在我们在告警、故障转移、故障定位及复盘时非常有用。

除了这些我们还要关注应用级的监控，我们的服务是否存活及健康、服务的质量如何。要明确的是不是进程存在服务就肯定存活，比如比较常见的接收线程占满的情况，无法对外提供服务，但进程还在，所有传统的通用进程判断的方式是不可取的。

服务存活判断最简单的方式是让服务提供ping接口，由监控服务定期ping这个接口，如有响应则表示存活。当然这有其局限，比如我们服务依赖了多个中间件，ping接口只是简单的请求-响应，无法感知中间件故障，所以一般的微服务框架都会提供health接口，此接口会报告当前服务的详细情况：

```json
{
    "status" : "DOWN",
    "db" : {
        "status" : "UP",
        "database" : "MYSQL"
     },
     "redis" : {
        "status" : "DOWN",
        "version" : "2.8.19"
     },
     "diskSpace" : {
         "status" : "UP",
         "free" : 309047318528,
         "threshold" : 20485760
     },
     "some-custom-service" : {
        "status" : "UP"
     }
}
```

这是返回示例，我们可以看到各中间件的情况，也可以将自定义服务加入到健康检查中，只要有一个服务DOWN了，服务整体的状态就标识为DOWN状态。这是一种非有效的服务存活及健康情况方式。

再深入些我们还会关心服务质量（Quality of Service），常见于特定接口的TPS、最大/90%/平均响应时间等性能指标，这些也需要服务提供特定接口暴露给监控服务。服务质量是判断是否需要动态伸缩的关键，后文会进一步说明。

对于核心的接口我们也需要针对性做监控，比如用户信息获取接口，这是大部分服务的公共依赖，非常重要，因此可以为这个接口做单独的请求-响应监控以便运维、开发可以在第一时间发现问题。

指标的采集是服务监控的第一步，主流微服务框架都提供了上述能力支撑，如需要自行集成推荐[micrometer](https://micrometer.io/) 这一工具，它也被Spring Boot 2.x做为默认的指标收集工具。

服务监控当然离不开运维工具，Zabbix、Nagios、Ganglia等都相当成熟，[Prometheus](https://prometheus.io/) 算是后期之秀，在云支持、微服务管理上有比较明显的优势。它是CNCF（Cloud Native Computing Foundation）成员，在对Kubernetes 集群环境下的监控有着先天优势。

特定框架也会有其自己生态下的监控工具，比如Spring Cloud的Spring Boot Admin，Dubbo的Dubbo Admin，但这些多是补充者的角色，无法替代主流运维工具。

![](https://raw.githubusercontent.com/gudaoxuri/Microservices-Architecture/master/resources/images/ms-montior1.png?sanitize=true)

如上图，是Spring Boot Admin的示例，它可以监控各服务的存活及健康情况，各个服务的详细信息，诸如JVM、访问日志、统计指标、环境配置、线程、实时接口调用、熔断等，另外还可以动态调整日志级别以快速排查问题。

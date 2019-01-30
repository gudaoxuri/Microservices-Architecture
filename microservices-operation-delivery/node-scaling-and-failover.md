# 节点伸缩与故障转移

容器化部署是微服务非常理想的选择，它可以实现上文提到的快速发布及本节要谈的自动化节点伸缩与故障转移。

自动化的节点伸缩也叫弹性伸缩，它在节点服务压力上升到指定阀值时自动增加一个或多个新的节点，在压力下降到指定阀值时又能自动回收节点，它的目标是最大化地控制资源利用率。自动故障转移指的是在某个节点或服务故障时可以自动启用备用节点或服务，确保服务实例数量大于等于设定的最小值，类似于分布式文件系统，如HDFS的副本策略。

要实现这两个目标的基础有2个：1）节点服务无状态，不持有数据（缓存除外），2）需要让运维系统感知到特定指标的值，这就用到了上一节的服务指标，节点伸缩需要的指标一般有CPU、内存、网络IO、JVM内存、核心接口TPS及平均响应时长，尤其是“核心接口TPS及平均响应时长”，这是最关键的指标，需要应用系统提供。故障转移最先判断是的节点的连通性，节点本身的硬件或系统故障、IP段所在交换机故障都有可能导致连接丢失，这时需要在与故障节点/网段尽可能远的地方（异地机房/其它机架等）启动备用服务，确定网络的连通性后还需要使用服务健康度相关指标，如果健康指标中反馈某个中件间有问题那要执行抢修（一般的中间件都是高可用的，如果应用服务连接中间件异常那多半是因为中间件发生了网络中断或比较严重的问题，中间件不像应用服务，它可能持有重要的数据，比较难实现故障转移），如果是某些应用接口故障，那同上文启动备用服务。

上述操作的核心在于运维工具上，目前与微服务最契合是工具当属Kubernetes，也称k8s，Kubernetes是Google开源的容器管理系统，同Prometheus一样，也是CNCF成员（Kubernetes是元老级的）。

Kubernetes可以很轻松地实现节点的伸缩，使用Horizontal Pod Autoscaler（HPA）来定时监控CPU或我们自定义的指标，如官网示例：

    kubectl autoscale rc foo --min=2 --max=5 --cpu-percent=80

设置foo这个控制器，HPA将通过增加或者减少Pod副本的数量（2到5之间）以保持所有Pod的平均CPU利用率在80%以内。自定义指标的伸缩可[参考](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/#support-for-custom-metrics)。Kubernetes默认支持连通性问题导致的故障转移，但健康度判断的故障转移需要自定义开发。

Kubernetes本身也实现或扩展了微服务的众多功能，如服务的注册发现、配置管理、负载均衡、调用跟踪等，这也使得有人提出了微服务使用Spring Cloud还是Kubernetes的讨论，实际上两者各有优劣，也都在彼此学习，比如[spring-cloud-kubernetes](https://github.com/spring-cloud-incubator/spring-cloud-kubernetes) 就是Spring Cloud与Kubernetes协同的集成方案，而[kubeflix](https://github.com/fabric8io/kubeflix)则是Kubernetes集成Netflix OSS的方案。

单从笔者的观点看以Kubernetes为代表的微服务架构试图让开发者用更简单的编程模型开发微服务化的系统是未来的趋势，本书最后会介绍的Service Mesh服务网格正是这种理念下的新生微服务架构。


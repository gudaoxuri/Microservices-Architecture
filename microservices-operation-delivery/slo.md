# SLO/SLA设定

SLO/SLA是什么？我们先一张图：

![](https://raw.githubusercontent.com/gudaoxuri/Microservices-Architecture/master/resources/images/ms-slo1.png)

上图是微软Azure虚拟机[SLA的部分截图](https://azure.microsoft.com/zh-cn/support/legal/sla/virtual-machines/v1_8/)，SLA（Service Level Agreement）服务等级协议是用于评估服务质量的重要标准，是合同性质，具备法律效力。任何正规的服务都会提供SLA，例如：
Azure所有服务的[SLA](https://azure.microsoft.com/zh-cn/support/legal/sla/)
阿里云ECS的[SLA](http://terms.aliyun.com/legal-agreement/terms/suit_bu1_ali_cloud/suit_bu1_ali_cloud201802011632_33742.html)
腾讯云[SLA](https://cloud.tencent.com/document/product/302/7506)
我们在比较各服务价格的同时也更需要关注他们的SLA，SLA的核心包含两块：服务的关键指标、未达要求的赔偿。

一般而言如果我们开发的是SAAS服务，那必需制定SLA，如果是内部服务那只要关心SLO就可以了。SLO（Service Level Objectives）服务等级目标是我们架构设计之初就要设定的，用于验收项目是否达到目标，也为SLA中的服务指标提供参考。

那么我们的SLO怎么设定？首先要明确下要有哪些指标，不同的产品SLO指标各不相同，基本上从可用性、性能、容量、安全等维度设计。

可用性是几乎所有产品都会关注的重要维度，笔者在架构设计时都会定义系统整体可用性、平均故障修复时间、最多停机维护次数/时间这几个指标。

![](https://raw.githubusercontent.com/gudaoxuri/Microservices-Architecture/master/resources/images/ms-slo2.png)

我们一般会用几个9表示可用性（High Availability），上图是wikipedia对几个9的说明，一般的云主机可用性要求是4个9即99.99%，这意味着全年只能有52分钟的故障时间。笔者所在平台服务部下的产品大多为99.95%，个别项目的特殊组件（比如微服务网关）要求达到99.999%。可用性整体地描述了全年最多允许的故障时长，平均故障修复时间（MTTR）则进一步限制了故障的平均修复时间。

性能也是我们很关注的维度，一般需要定义TPS(Transaction Per Second)、最大/平均响应时间(latency)等，这是衡量系统性能的关键指标。

容量维度主要是定义产品最多可支撑的用户量、同时在线量、最大数据处理体量等，安全维度关注数据是否丢失、数据权限是否有问题等。

有确定的SLO指标后我们面临的最大问题是有些值怎么确定，而这就与我们微服务架构紧密相关了。比如我们对TPS的设定：

![](https://raw.githubusercontent.com/gudaoxuri/Microservices-Architecture/master/resources/images/ms-slo3.png)

已知接口2、接口3、接口4没有其它服务依赖，所以我们通过压力测试可以得到各个接口实例的TPS，那接口1依赖接口3和接口4，它的TPS是多少？服务A整体的TPS又是多少？有同事告诉我可以测算接口1调用接口3和接口4的占比进而估算接口1的TPS，服务A的TPS同理估算。但问题在调用占比可以测算，但响应时间无法测算，即使接口1只有1%的调用依赖于接口4，但如果这些都是长耗时请求，那么接口1的TPS会更接近接口4的TPS而非接口3。所以如果想测算有依赖接口的TPS并不容易。笔者的经验是只能是估算+整体的压测确定一个相对保守的值。

有同样的问题还有我们的可用性，可用性不适用于木桶理论，不能简单将可用性最低的服务作为产品的可用性指标，服务要在业务上分级，针对业务重要性及服务可用性做一定的加权处理。比如：`产品可用性 =（核心服务A可用性95%*10+边缘服务B可用性60%*1）/ 11=91.8%` 。当然这也只是估算，权限值的设定也多半由经验所得。

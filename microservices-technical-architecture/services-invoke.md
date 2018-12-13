# 编制与协同设计

服务治理这个词相信很多读者都听过，而其核心在于对服务调用的管理，这里就涉及到服务编制（[Orchestration](https://en.wikipedia.org/wiki/Orchestration_(computing)) ）与协同（[Choreography](https://en.wikipedia.org/wiki/Service_choreography) ，也有称编排），两者很容易混淆，中文翻译更是词不达意（如Google翻译将Service Orchestration和Service Choreography都翻译成服务编排，百度翻译Service Choreography为服务编舞），本书使用编制与协同是为更好区别两者，为明确意图下文直接使用英文单词。

![](https://raw.githubusercontent.com/gudaoxuri/Microservices-Architecture/master/resources/images/ms-services-invoke-orchestration-choreography.png?sanitize=true)

单从字面看Orchestration为管弦乐曲，由一人负责指挥与调控各个乐师，而Choreography为舞蹈，各个舞蹈者多为对等关系，没有统一的协调者。事实上也是如此，Orchestration需要类似ESB的服务总线来统一管理调度各个服务间的通讯，而Choreography更强调的是各服务自治，各自自己去调用需要的服务，相对而言更去中心化。

>🔆 ESB（Enterprise Service Bus）原指企业消息总线，是SOA重要的组成，这里借用下这一概念，在微服务中，ESB可以更轻量，比如Nginx或是Zuul等。


我们可以看出Orchestration是有中心化调度服务以协调各组件通信的方式，以用户注册流程为例：

![](https://raw.githubusercontent.com/gudaoxuri/Microservices-Architecture/master/resources/images/ms-services-invoke-register-orchestration.png?sanitize=true)

各服务间都通过ESB进行数据通信，用户注册后会向ESB发送调用活动服务的用户注册接口，ESB会将对应的请求路由给活动服务，活动服务收到请求并处理后又向ESB发送调用卡券服务生成卡券，调用积分服务生成积分，ESB路由此请求给卡券服务和积分服务。

我们可以在ESB中做集中式的权限管控、日志处理等增强操作，这是它很大的优势，但也带来了不少的问题，存在中心化的ESB，所有请求都要经由ESB，所以在性能、扩展性、灵活性上会比较差。

相反，Choreography则是去中心化的、点对点的通信的方式，还是以注册为例：

![](https://raw.githubusercontent.com/gudaoxuri/Microservices-Architecture/master/resources/images/ms-services-invoke-register-choreography.png?sanitize=true)

所有的请求都是直接调用对应的服务，没有ESB，这带来的直接好处是性能的提升，但它与Orchestration一样存在一定的服务耦合，比如用户服务就需要感知到卡券服务及活动服务，活动服务需要感知到卡券服务，我们可以引入事件架构（见下一节）以避免这一问题。

从微服务所面对的场景分析，服务协同更为合适，这也是微服务下主流的服务治理方案。










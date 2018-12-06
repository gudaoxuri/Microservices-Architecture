# 服务划分

服务的合理划分，即服务边界的设定是微服务成功的重中之重，是所有项目实施之前必须认真思考，严肃对待的。

一个合理的服务划分应该是：

* **符合团队结构** 服务的落地与维护靠人，靠的是执行团队（包含业务、产品、技术、测试与运维团队），所以服务的设定一定是与团队结构相辅相成的，同一个系统不同的执行团队往往会有不同的且都合理的服务划分方案
* **业务边界清晰** 各服务有清晰的责任及边界，一个服务对应一块业务，服务间多为单向依赖
* **最小化地变更** 新增或变更业务上有很明确的服务对应，或是新增服务或是扩展某些服务，很少出现既可以在这个服务上实现也可以在那个服务上实现这种摸棱两可的情况，在符合上述条件前提下某一业务需求地变更受影响的服务应该尽可能地少
* **最大化地复用** 服务的复用是服务化的一个重大优势，服务设定要考虑复用的场景，在符合上述条件的前提下应该尽可能最大化地实现服务复用
* **性能稳定简洁** 上述条件更多的是业务导向，从技术上服务设定的核心要关注对性能的影响、是否稳定及架构是否简洁，是否要引入额外的中间件等

那么我们应该怎么做才能满足这些要求呢？下文笔者就带领各位一同探讨下这个问题。

## 以业务、技术、团队导向规划服务

我们必须明确的是：服务不是越细越好，服务划分的第一要素是先以业务域拆分，再以技术视角拆分，结合团队的规模、能力确定服务间的关系与边界。

![](https://raw.githubusercontent.com/gudaoxuri/Microservices-Architecture/master/resources/images/ms-services-division1.png?sanitize=true)

对车贷通而言，上图是它的相对原子化的业务服务单元，理想情况以此拆分可到较细粒度的服务，但考虑两点：

* 业务上，一般而言一个需求最好只影响一个服务，这样可保证快速版本迭代，但贷款申请、审核、放款是一个完整流程的操作，如果申请中修改了填报的信息，那在审核、放款时也必定要处理或展现，所以对贷款流程的需求可能需要修改三个服务，这明显不合理
* 团队上，成员规模及技能水平有限，过细的拆分会导致开发、维护困难

所以我们重新划分下边界：

![](https://raw.githubusercontent.com/gudaoxuri/Microservices-Architecture/master/resources/images/ms-services-division2.png?sanitize=true)

我们划分成四个服务：基础服务包含客户信息管理、金融产品管理及用户权限管理，所有贷款流程相关的都放在贷中服务，还款、客户维系、催收等划到贷后服务，论坛独立成一个服务。这样服务边界更清晰了，但是有如下几个问题：

* 所有服务都会依赖用户权限管理
* 登录、注册、催收等都需要发短信

可见存在一些服务是公共的，针对这个情况我们可以再以技术视角做垂直拆分：

![](https://raw.githubusercontent.com/gudaoxuri/Microservices-Architecture/master/resources/images/ms-services-division3.png?sanitize=true)

现在我们分别从业务域及技术域上做了简单的服务边界划分，看似符合要求了，但从架构的全局观上分析还是有比较大的问题，主要体现在对业务规划缺乏理解，在业务成熟后存量客户的维系及新客户的拓展会成为一个重心，一般而言会有配套的精确化营销（或类似）系统，同期数据化经分系统也会成为核心以为各类决策提供数据支撑，上述架构没有体现出业务系统边界，因此我们再次修正下：

![](https://raw.githubusercontent.com/gudaoxuri/Microservices-Architecture/master/resources/images/ms-services-division4.png?sanitize=true)

团队的整体能力是要考虑一个重要因素，一般而言团队的整体能力与服务的数量成正比，反之极容易导致架构失控。

## 领域检查

现在看起来边界更清晰：明确了系统构成及各系统内的服务，但别急。由于我们这个示例比较单一，所以各系统的业务域及服务的业务边界比较明确，但在有多条产品线、多项目团队合作研发的场景下极容易出现某些功能看上去在哪个产品中做都可以的情况，这时如何划分边界就比较棘手了。

这种情况下我们可以引入领域模型。使用领域模型为服务的业务划分提供指导是个很好的开始，国内也有不少优秀的实践[Jdon](http://www.jdon.com/)。

>🔆 领域驱动设计：DDD（Domain-driven design）是一套综合软件系统分析和设计的面向对象建模方法。[详见此处](https://en.wikipedia.org/wiki/Domain-driven_design)

下图笔者所服务公司资产端数据结构的领域模型示例，通过领域建模可以很清楚地明确各实体对象的归属，进而确定业务服务的边界。

![DDD示例](https://raw.githubusercontent.com/gudaoxuri/Microservices-Architecture/master/resources/images/ms-services-division-ddd.png?sanitize=true)

领域建模对服务的划分有非常重要的指导意义，即使我们不用DDD也应该多少了解领域模型设计，反观国内IT企业大家对这块太不重视了，举一个例子：某国内知名的垂直电商公司的CTO公开讲他们订单与优惠券的设计演进，第一个版本核心结构如下：

![](https://raw.githubusercontent.com/gudaoxuri/Microservices-Architecture/master/resources/images/ms-services-division-ddd-analysis1.png?sanitize=true)

这一版很简单也很好理解，问题在于它将优惠券与商铺绑定，但优惠券有针对商品的、商铺的和平台的，比如X商铺A商品5折、X商铺满100减20，平台满200减20等，上面的结构明显不符合要求，所以他们改成如下结构：

![](https://raw.githubusercontent.com/gudaoxuri/Microservices-Architecture/master/resources/images/ms-services-division-ddd-analysis2.png?sanitize=true)

做法是将订单拆分成针对商品、店铺及平台（支付）的三类，一个支付订单可以包含一个或多个店铺订单，一个店铺订单可以包含一个或多个商品订单，问题解决了吗？的确从功能上看是满足了，但这种做法的后患是订单与优惠券完全绑定了，订单被优惠券绑架了，如果业务上又出现了针对不同类目的优惠（这很常见）那是不是又要加入类目级订单？

这就是典型的缺乏领域建模思维的产物，如果用领域建模会是怎样？从领域角度看这种做法是完全错误的，订单与优惠券分属于两个领域，一个是核心域（暂且这么称呼），另一个是运营活动域，原则上运营活动域是可选的，不应该为支撑可选域而对核心域做过多的侵入（从1个订单变成3个订单），所以理想情况下优惠券对订单的支持应该只是在运营活动域中处理。理解了这个后模型很简单：

![](https://raw.githubusercontent.com/gudaoxuri/Microservices-Architecture/master/resources/images/ms-services-division-ddd-analysis3.png?sanitize=true)

引入一张订单优惠券的关联表（Order Coupon）即可，核心域只关注订单，各类活动的处理在运营活动域中操作。

就这么简单？是的，从领域处理上就是这样，但要支持我们的需求需要有些特殊的处理，在关联表要引入事务ID，同一次操作事务ID相同。相关的操作逻辑如有兴趣欢迎到互动论坛中留言讨论。

领域检查可以为我们的服务划分提供方向性的指导，使服务划分更明确、各有规划性。








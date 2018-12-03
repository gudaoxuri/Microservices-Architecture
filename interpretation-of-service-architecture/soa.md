# SOA的演进

在了解到单体架构的这些不足后大家会自然而然地想到做如下的拆分：

![&#x5355;&#x4F53;&#x62C6;&#x5206;](https://raw.githubusercontent.com/gudaoxuri/Microservices-Architecture/master/resources/images/monolithic-split.svg?sanitize=true)

我们可以从业务上将不同的模块拆分到不同的系统中，模块1、模块2划到A系统，其它的模块划到B系统， 我们还可以将同一模块进一步拆分，比如模块3有Controller层和Service层，那就可以按技术分层将一个模块拆分成多个。 而无论我们怎么拆分都会面临一个问题：拆分后的服务怎么调度？这时就该介绍一下SOA了。

![SOA](https://raw.githubusercontent.com/gudaoxuri/Microservices-Architecture/master/resources/images/soa.svg?sanitize=true)

🔆 SOA（Service-Oriented Architecture）是一种分布式服务架构的常见方式：提供一种被各个服务单元/系统彼此认可的协议进行数据通讯，进而实现跨服务单元/系统交互的能力。 [详见此处](https://en.wikipedia.org/wiki/Service-oriented_architecture)

SOA在软件工程及IT行业的发展中有着举足轻重的地位，在系统标准化、集成化、规模化建设中起到了关键作用。它的核心优势是以相对简单的方式解决了系统间的服务重用问题。为各个独立的、松散的系统互通提供了解决方案。

早期的SOA多以基于SOAP \(Simple Object Access Protocol\)协议的Webservice实现，因其通过WSDL \(Web Service Description Language\)实现了严格服务间通讯格式约束，所以它非常适用于对稳定性要求高、不易变、防御式的场景， 这种方式在当下的银行、电信系统中还有很大市场。但SOAP的优势也为其带来了：笨重、不灵活、修改困难、通讯效率低等问题。

SOAP怎么用？大的流程如下（详细说明参见 ）：

1. 先定义服务锲约，如国家信息查询服务（countries.xsd）

```markup
<xs:schema xmlns:xs="http://www.w3.org/2001/XMLSchema" xmlns:tns="http://spring.io/guides/gs-producing-web-service"
           targetNamespace="http://spring.io/guides/gs-producing-web-service" elementFormDefault="qualified">
    <!—请求的名称、参数-->
    <xs:element name="getCountryRequest">
        <xs:complexType>
            <xs:sequence>
                <xs:element name="name" type="xs:string"/>
            </xs:sequence>
        </xs:complexType>
    </xs:element>
    <!—对应响应的名称及参数-->
    <xs:element name="getCountryResponse">
        <xs:complexType>
            <xs:sequence>
                <!—参数是自定义类型-->
                <xs:element name="country" type="tns:country"/>
            </xs:sequence>
        </xs:complexType>
    </xs:element>
    <!—自定义类型结构-->
    <xs:complexType name="country">
        <xs:sequence>
            <xs:element name="name" type="xs:string"/>
            <xs:element name="population" type="xs:int"/>
            <xs:element name="capital" type="xs:string"/>
        </xs:sequence>
    </xs:complexType>
</xs:schema>
```

1. 使用工具（如jaxb2-maven-plugin）将xsd生成对应的Java并初始化数据（略）
2. 将此服务以SOAP协议提供（如使用spring-boot-starter-web-services）（略）
3. 测试

```markup
# HTTP提交查询内容：
<soapenv:Envelope xmlns:soapenv="http://schemas.xmlsoap.org/soap/envelope/"
                  xmlns:gs="http://spring.io/guides/gs-producing-web-service">
   <soapenv:Header/>
   <soapenv:Body>
      <!—调用getCountryRequest 方法，查询国家是西班牙-->
      <gs:getCountryRequest>
         <gs:name>Spain</gs:name>
      </gs:getCountryRequest>
   </soapenv:Body>
</soapenv:Envelope>

# 服务返回结果
<?xml version="1.0"?>
<SOAP-ENV:Envelope xmlns:SOAP-ENV="http://schemas.xmlsoap.org/soap/envelope/">
  <SOAP-ENV:Header/>
  <SOAP-ENV:Body>
    <ns2:getCountryResponse xmlns:ns2="http://spring.io/guides/gs-producing-web-service">
      <ns2:country>
        <!—西班牙的信息-->
        <ns2:name>Spain</ns2:name>
        <ns2:population>46704314</ns2:population>
        <ns2:capital>Madrid</ns2:capital>
      </ns2:country>
    </ns2:getCountryResponse>
  </SOAP-ENV:Body>
</SOAP-ENV:Envelope>
```

REST（Representational State Transfer）的出现很好地解决了上述问题，它将所有服务抽象为资源，资源不仅代表服务器中的文件、数据库中的表等具体的东西，它可以是任何对象。 通过URL定位资源，使用 HTTP 方法（HTTP Method）确定对资源执行什么操作，通俗地说（严格而言有一定问题，见下文说明）所有操作都可以抽象为CRUD（Create-创建，Retrieve-查询，Update-更新，Delete-删除），可分别对应于HTTP的 POST、GET、PUT、DELETE。 由于使用标准的HTTP操作、没有Schema约束，REST成为系统间轻量交互的首选方式。

REST的核心特征是面向资源（Resource Oriented）、可寻址（Addressability）、连通性（Connectedness）、无状态（Statelessness）、统一接口（Uniform Interface）、超文本驱动（Hypertext Driven）。 [详见此处](http://www.infoq.com/cn/articles/understanding-restful-style)

上文SOAP的方式换成REST时对应的流程为：

1. 定义一个HTTP服务，以Spring MVC为例：

```java
@RestController("/country")
public class CountryController {
    @GetMapping("")
    public CountryInfo getCountryInfo(@RequestParam("name") String name) {
        // 查询对应的国家信息
        return ...;
    }

    public static class CountryInfo{
        // ...
    }
}
```

1. 测试

```bash
# HTTP提交查询内容：
curl --header "content-type: application/json" http://localhost:8080/country?name= Spain

# 服务返回结果
{
    name:Spain,
    population: 46704314,
    capital: Madrid
}
```

> 🔆 REST 可重新表达的状态迁移（Representational State Transfer）是Roy Thomas Fielding博士于2000年在他的博士论文中提出来的一种互联网软件架构风格， 目的是便于不同软件/程序在网络中互相传递信息。 使用上看似简单，但背后有着深厚的理论支持。 [详见此处](https://en.wikipedia.org/wiki/Representational_state_transfer)
>
> 📖 **关于HTTP方法的说明**
>
> 一般而言我们都认为HTTP POST用于创建资源，PUT用于更新资源，但严格意义两者在区别在于是否幂等，它们都可以表示创建或是更新资源，POST不保证幂等，PUT要求保证幂等，区别仅此而已 \[[understanding\_rest](https://spring.io/understanding/REST)\] \[[put-vs-post](https://stackoverflow.com/questions/630453/put-vs-post-in-rest)\]。 基于此定义我们可以认为资源的唯一标识由请求方设定时可使用PUT创建，以保证多次请求的幂等，反之使用POST。 PUT操作需要注意的是与其说是更新资源倒不如说是替换资源更为贴切，一次PUT的更新操作要求请求方拥有资源的完整信息，将更新存在资源的所有字段，如果要实现部分字段更新应使用PATCH方法，PATCH非幂等。 另外POST也可用于查询，不同浏览器对URL的长度有限制并且查询条件放在URL中存在一定的安全隐患，所以在一些场景下会用POST代替GET实现对资源的查询操作。
>
> 📖 **REST与HTTP的关系**
>
> HTTP是REST的一个主要表现协议，但REST本身是协议无关的（REST论文也只提到REST在HTTP下的应用\[[resp-applied-to-http](https://www.ics.uci.edu/~fielding/pubs/dissertation/evaluation.htm#sec_6_3)\]），也更没有规定HTTP方法与资源操作的映射关系\[[soap-vs-rest-differences](https://stackoverflow.com/questions/19884295/soap-vs-rest-differences)\]，所以严格意义上我们要明确两者的关系，但在实际项目及开发中可近似地认为REST就是用于HTTP的。

REST流行之初曾因其没有协议强约束而有过与SOA的对比讨论\[[can-a-soa-be-designed-with-rest](https://stackoverflow.com/questions/10491812/can-a-soa-be-designed-with-rest?utm_medium=organic&utm_source=google_rich_qa&utm_campaign=google_rich_qa)\]，但SOA本身也在不断演化，以现在的观点看和SOAP一样，REST不过是Webservice的另一种协议，自然也是属于SOA大家庭的一员。

> ⚠ 我们花了些篇幅介绍SOA只因为它太重要了，由于篇幅所限，关于SOA深入的知识及不同时间段人们理解的分歧在脚注中给了链接，有兴趣的读者是可作为扩展阅读进一步加深了解。

随着IT地发展，人们对软件产品的依赖度越来越高，这使得现代软件研发规模化、规范化、可快速迭代。传统的（为什么说传统的？见下文）SOA面临着前所未有的挑战，概括而言如下：

* **传统SOA拆分的粒度较大** 一般按业务域划分系统，但很少涉及系统内细粒度地拆分。一个系统能有多大？一个普通的系统百万级别的代码、几十人的研发团队是司空见惯的，传统SOA只解决了系统的交互，但没能从架构层面解决系统内的复杂度、效率、安全等单体架构所存在的问题
* **传统SOA多需要集中的服务总线，容易产生性能瓶颈** ESB（Enterprise service bus）几乎是传统SOA必备的，它集通信交互、服务编排、认证授权、质量监控等功能于一身，可十分方便地管理各个零散的系统。Oracle、Microsoft、IBM都有成熟的商业产品，也有诸如wso2、Apache Synapse等开源免费的产品，很多有一定能力的公司也会自研ESB， 一度时间ESB市场兴兴向荣，但它的问题在于随着接入系统的增多、系统间调用频率的增加ESB成为了制约产品性能提升的瓶颈，并且ESB的稳定性直接影响了产品的质量。 如果我们将系统再拆分成一个个服务，那么ESB的接入方将增加十倍、百倍，并且系统内各服务的调用频次远高于系统间，那时ESB的压力会增加百倍、千倍。因此ESB这种集中化的服务调度、管理方案必定需要优化。

当我们的服务上规模后以上问题会被放大，变得不可忽视，那么我们究竟需要一个怎样的架构呢？


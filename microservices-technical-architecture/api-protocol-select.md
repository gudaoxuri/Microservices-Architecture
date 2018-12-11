# 接口协议选择

确定了服务划分微服务改造算是迈出了关键的一步，接下来我们要考虑选择合适的接口协议以实现服务间的数据通信。

目前主流的接口调用方式有两大类：RPC与Rest。

>🔆 RPC(Remote Procedure Call): 以本地方法调用的形式处理远程方法调用的模型。常用的有：SOAP、RMI、Thrift、Avro、gRPC、Dubbo协议。

>🔆 REST(REpresentational State Transfer)：以资源为中心，描述资源状态变更的模型。常见于HTTP协议。（上一章有比较详尽的描述）

RPC是Bruce_Jay_Nelson在1984年的[《Implementing Remote Procedure Calls》](http://birrell.org/andrew/papers/ImplementingRPC.pdf)文章中首次提出的，用于构建简单、高效、通用的通信机制。以gRPC、Dubbo为代表的RPC方式的优点有：

* 可定制协议/传输类型，可实现高性能通讯
* 可用于强格式约束场景


使用RPC一般而言要先定义IDL（Interface description language，接口描述言），以gRPC为例，我们需要定义类似的proto文件：

```
// 定义服务名称
service Greeter {
  // 定义方法
  rpc SayHello (HelloRequest) returns (HelloReply) {}
}

// 方法入参消息体
message HelloRequest {
  string name = 1;
}

// 方法出参消息体
message HelloReply {
  string message = 1;
}
```

对于Java而言，可以使用protobuf-maven-plugin 这一Maven插件将上述代码编译成Java文件，之后在服务端就可以实现对应的方法，如：

```java
private class GreeterImpl extends GreeterGrpc.GreeterImplBase {

  @Override
  public void sayHello(HelloRequest req, StreamObserver<HelloReply> responseObserver) {
    HelloReply reply = HelloReply.newBuilder().setMessage("Hello " + req.getName()).build();
    responseObserver.onNext(reply);
    responseObserver.onCompleted();
  }

}
```

具体示例官方都有，这里不赘述，当然我们也可以使用不同的工具生成C++、NodeJS、C#、Go等不同的语言的实现。

由于Dubbo只支持JVM平台，所以它的IDL只要定义Java的接口即可。

所以，我们不难看出RPC存在的不足包含：

* 各调用方多有语言约束，以Thrift、Avro、gRPC为代表的跨语言RPC需要定义IDL，以Dubbo、RMI为代表的无需显示指定IDL的RPC又对协议各方有语言限定
* 字段变更各调用方多需要重新部署，常见的，当有新的服务调用方参与要重用某一接口的同时又需要为其添加新的字段时必须修改IDL，大部分的RPC框架都要求所有调用方同步更新

我们再看下REST，它多基于HTTP协议实现，优点有：

* 通用性高，无语言约束，主流的语言都对其提供了很好的支持
* 弱格式约束，字段变更不需要所有调用方都重新部署
* 防火墙友好，HTTP做为最普遍的协议更容易被防火墙接受

REST一般会基于Json或XML做为交互格式，两者均为跨语言、弱约束的格式。我们不需要先定义IDL再生成对应的语言文件，接口的变更除受影响的参与方需要修改外不需要全局重新部署，例如：

```
GET /user/{id}
{
  "name":""
}
```

这是我们的获取用户信息接口，返回用户姓名，有4个业务方调用，但后来有其中一个业务方要求再返回年龄信息，那么我们接口可以修改成：

GET /user/{id}
{
  "name":"",
  "age":0
}

这里增加了age字段只为某一业务方使用，其它业务方可以不用同步修改。在实际生产中我们一般会为核心接口增加版本号字段以更好地做多版本兼容。

为REST API添加版本有三种方式：

* 版本信息放在Path或Query中，如

```
api.example.com/v1
api_v1.example.com
api.example.com/xxx?version=v1
```

* 使用自定义Header，如

```
Accept-version：v1
Accept-version：v2
```

* 使用内容协商（Content negotiation）

```
Accept: application/vnd.example.v1+json
Accept: application/vnd.example+json;version=1.0
```

从REST设计的初衷而言，URL对于表述资源而与版本无关，因此纯学术的建议是使用第2、3种方式，但我们也看了大量第1种方式的API，其中不乏有知名的IT厂商，所以笔者觉得这3种都可以，如何选择完全看团队的风格。[更多讨论](https://stackoverflow.com/questions/389169/best-practices-for-api-versioning)。

在引入版本后接口锲约的向下兼容性可以放开，相对更自由，新的业务修改只要升版本即可，但这对服务多版本维护及兼容提出了更高的要求，实践中如果服务逻辑变更很大可以为新版本发布一个对应的新服务，如果变更相对有限则多半可以在服务中增加兼容/适配层来解决。

当然REST也有自身的局限，比如传输效率没有特定的RPC框架高，由于没有强契约规范，对字段、结构的修改可能会导致已接入调用方的异常。

那么我们究竟如何选择呢？以微服务架构的要求看更偏向于使用REST以方便实现异构系统的通讯。笔者对比REST的代表Spring Cloud与RPC的代表Dubbo，IO性能上前者比后者慢1/2左右，但对于实际业务调用场景而言，大部分情况下通讯上的这些损失是可以接受的，并且我们还可以用缓存、压缩等方式进一步减少两者的差距。对于接口修改可能影响已接入调用方的问题需要我们通过一定的开发规范加以防范，比如常见的，在没有版本的情况下我们不允许修改、删除字段，如果某字段的定义变更或已有的字段不能满足需要则使用新增字段解决。对于部分业务相对稳定的服务，尤其是一些核心的高频调用的公共服务，我们可以考虑使用RPC以提升效率并强制锲约检查，可惜的目前并没有兼容两种调用类型的微服务架构，Spring Cloud社区有第三方插件支持gPRC，有兴趣的读者去Github上关注。

就车贷通而言，我们整体的并发要求不高，核心接口TPS不会超过1000，所以选择REST是比较自然的决定。在后期版本的迭代中可以考虑将部分核心服务改成gRPC以提升性能及稳定性。




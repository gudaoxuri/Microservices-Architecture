# 接口管理

接口（API）是服务能力的包装及体现，是服务提供的直接途径，也是做为服务使用方最关注的方面，一个设计合理、稳定不易变、文档清晰的接口集合是服务质量重要组成。

## 设计合理

怎样称得上“设计合理”？笔者认为需要满足以下几点：

* **能力覆盖全** 接口集合要尽可能地体现服务能力，以邮件服务为例，邮件的发送包含主题、正文（正文又分为纯文本与富文本）、收件人、抄送人、暗送人、附件……等，那么对应的接口也应该包含这些参数，同时诸如已读回执、退信通知等功能也应该考虑在接口之中

* **功能单一** 一个接口只对应一个能力，类同设计模式中的单一责任链，这是绝大部分接口都要遵守的，但也有例外，比如在消息服务的设计中笔者就将短信、PUSH、站内信、微信、钉钉等发送能力集成在了同一个接口中，而其底层实则对应了多种服务

* **简单易用** 接口的设计要遵循迪米特法则，只暴露必要的接口及参数，再拿上面消息服务举例，短信会对接多个提供商，不同提供商的价格、服务范围、服务质量不同，如果将提供商的选择放在发送接口中就会给使用方造成困扰，所以更合适的做法是内部消化掉，我们可以将之与发送模板绑定，发送时只要选择对应的模板即可

* **格式规范** 一个服务接口集合的格式应该更可能地规范，这包含
    * 统一格式，要么都是json，要么都用xml
    * 命名合理，业务指向明确，如 `GET /management/message/{id}` 与 `GET /message/{id}` 可以明确地区别前者是管理侧使用，如果是restful，那么严格遵循其约定（详见上文），比如添加一条记录如果是幂等时应该用`PUT`而不是`POST`

## 稳定不易变

需求的迭代过程中不可避免地会发生接口的变更，而微服务下更是会将大接口拆分成一个个小接口导致接口数量成倍增加，并且微服务的调用链路普遍长于传统架构，一个接口的变更可能会级联很多业务的修改，所以如何处理接口的变更是非常值得讨论的问题。我们常用方法有：

* 通知所有消费者修改，某个接口变更成这样，大家赶紧改一下
* 接口加上版本，变更接口会新增版本，老版本的接口不受影响
* 新增一个接口，原接口不变
* 字段只增不改，维持原接口各字段的口径定义不变，需求的变更只增加字段

理想情况下，字段只增不改是最优选择，它体现了Rest相较于RPC在弱锲约下的优势，原有的消费者完全不用修改即可适配新的锲约，应对常规性的需求变更这会是比较合适的做法，但它的问题也比较明显，它不能修改字段可能会导致在不断地版本迭代中产生越来越多的垃圾字段，比如获取用户信息的接口中有地址信息，最初版本就是`address`一个字段，第二个版本为适配新的细分地址需求新增`province、city、county、detail`等字段，第三个版本要求使用统一的行政编码于是新增了`provinceCode、cityCode、detailCode`等字段，第四个版本要区分是住址还是工作地址于是又新增了`live.provinceCode、live.cityCode、live.detailCode、work.provinceCode、work.cityCode、work.detailCode`，那么在一段时间后我们反观最初版本的`address`及后来的`province、city、county`可能就没有消费者使用进而成为了垃圾字段，所以在几次迭代后我们要清理这些没被使用的字段以使我们的接口可以放下历史包袱，更加简洁可读。

怎么做？我们可以引入版本，用老版本做兼容，用新版本处理需求的变更，这也是很多成熟的平台服务所采用的方案，此方案的成本在于要同时维护多套逻辑以应对不同的版本，在投入与产出上需要权衡，如果条件允许这会是最“稳妥”的方案，提升服务的可信赖感。

那么什么时候需要新增加一个接口呢？很多情况锲约变更比较大就新会增一个接口，常见于`GET: /user/{id}`到 `GET: /user-new/{id}`，且不论这样命名是否合理，单就新增接口本身而言就是错误的，新增接口只发生在接口的定义发生变更而不在于原接口契约的变更程度，比如`GET：/user/{id}/overview` 和 `GET:/user/{id}/detail`这两个接口就有比较清晰的定义区别。随意地新增接口不但会让消费者产生困惑更会导致服务的不可控。

在某些情况下我们会不得以去为没有版本化的接口修改锲约，此时就需要通知所有的消费者，这是比较危险的操作，既需要能通知到所有消费者又要保证所有消费者在同一时间修改，具有比较大的升级成本及很高的不可控因素，如非必须切勿采用这一方案。如需要执行此操作我们前期就要做好消费者的记录，升级前预留足够的时间窗口以让所有的消费者收到通知并有时间修改自身的逻辑。

接口是产品对外服务的重要途径，保持接口定义的严谨性、稳定性是产品质量的重要组成部分，我们应该在产品团队的能力范围内尽可能地做到平滑升级。

## 文档清晰

接口文档比较特殊，它与我们的代码实现紧密相关，如果脱离代码手工撰写会导致项目越演进接口文档与代码越不匹配。所以接口文档与代码集成，或是说让代码自我描述成文档是最理想的选择，而这块最为成熟的当属于`Swagger`。

Swagger是强大且成熟的API管理工具，可与主流框架结合简单地生成优雅的在线文档，以Spring Boot为例我们在对应的Controller中加入接口信息：

```java
@RestController
@Api(value = "/apply", description = "申请接口") // Swagger 接口集合 声明
@RequestMapping(path = "/apply")
public class ApplyController {

    @PostMapping(value = "/credit")
    @ApiOperation(value = "提交授信申请")         // Swagger 某一接口 声明
    public Resp<CreditApplyResp> submitCreditApply(@RequestBody CreditApplyReq creditApplyReq) {
       // TODO
    }

    @GetMapping(value = "/credit/{id}/status")
    @ApiOperation(value = "查询授信申请状态")       // Swagger 某一接口 声明
    @ApiImplicitParams({
            @ApiImplicitParam(name = "id", value = "申请Id", paramType = "path", dataType = "long", required = true),                       // Swagger 某一接口 声明
    })
    public Resp<CreditApplyResp> getCreditApplyStatus(@PathVariable("id") long creditId) {
        // TODO
    }

}
```

在DTO中也添加相应的模型及字段说明：

```java
@ApiModel("授信申请请求")
public class CreditApplyReq {

    @ApiModelProperty(name = "姓名",required = true)
    private String name;

    @ApiModelProperty(name = "身份证",required = true)
    private String idcard;

    @ApiModelProperty(name = "手机",required = true)
    private String phone;

    @ApiModelProperty(name = "关联产品",required = true)
    private long productId;

   ...
}

@ApiModel("授信申请响应")
public class CreditApplyResp {

    @ApiModelProperty(name = "申请Id", required = true)
    private long id;

    @ApiModelProperty(name = "申请身份证", required = true)
    private String idcard;

    @ApiModelProperty(name = "申请状态", required = true, allowableValues = "PROCESSING=处理中,SUCCESSFUL=成功,FAILURE=失败")
    private String status;

    @ApiModelProperty(name = "状态查询时间戳", required = true)
    private long queryTime;

    ...
}
```

最后添加相应的Bean配置全局的Swagger参数：

```java
@Bean
public Docket restApi(ServletContext servletContext) {
    return new Docket(DocumentationType.SWAGGER_2)
            .apiInfo(apiInfo())
            .select()
            .apis(RequestHandlerSelectors.basePackage(...))
            .paths(PathSelectors.any())
            .build()
            .pathProvider(new RelativePathProvider(servletContext) {
                @Override
                public String getApplicationBasePath() {
                    return contextPath + super.getApplicationBasePath();
                }
            });
}

private ApiInfo apiInfo() {
    return new ApiInfoBuilder()
            .title(...)
            .description(...)
            .termsOfServiceUrl(...)
            .version(...)
            .build();
}
```

这样我们就可以通过 /swagger-ui.html 来访问了，示例如下：

![](https://raw.githubusercontent.com/gudaoxuri/Microservices-Architecture/master/resources/images/ms-api-management1.png)

我们可以直接在界面上进行测试，如：

![](https://raw.githubusercontent.com/gudaoxuri/Microservices-Architecture/master/resources/images/ms-api-management2.png)









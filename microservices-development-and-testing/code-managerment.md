# 代码管理

上文讲的是人员和团队，提出要将团队划小管理，那具体到开发层面，我们如何建立与之对应的工程结构及版本管控呢？

在我们传统架构下一个项目一般对应于一个或几个版本库（服务端一个、几个大前端，如Android、IOS、Web），很少有项目会将服务端代码按服务域或子系统拆分到不同的版本库中。但微服务因其服务多且服务间由不同团队开发，无论从代码安全及上文讲的团队独立等因素我们在设计工程结构时都应该充分考虑服务间的隔离。

版本管理最优选择必定是Git，我们可以用Git的Submodules实现上述要求。一份笔者常用的工程结构，一个工程对应一个Git库如下：

```
X Build Project                 // 构建工程
|- sources                      // 源码目录
|   |- basics                   // 基础源码，为各服务工程的依赖，要求deploy到Maven仓库
|   |   |- parent               // 父工程，所有服务端项目的根POM
|   |   |- common               // 公共工程，所有服务端项目的基础依赖
|   |   |- common-service       // 公共服务工程，所有服务工程的根POM，自身依赖于common
|   |   |- <...>                // 其它基础工程，比如common-spark，大数据服务的基础依赖
|   |- services                 // 服务源码，对应于一个个微服务
|   |   |- <service 1>          // 服务1工程
|   |   |- <service ...>        // 其它服务工程
|   |- sdk                      // SDK源码，如项目需要提供SDK时建立此目录
|   |   |- <java>               // Java版本的SDK工程
|   |   |- <js>                 // JS版本的SDK工程
|   |   |- <rest>               // REST版本的SDK工程
|   |   |- <...>                // 其它语言版本的SDK工程
|   |- terminals                // 终端源码
|   |   |- <android>            // Android APP工程
|   |   |- <ios>                // IOS APP工程
|   |   |- <wechat>             // Wechat工程
|   |   |- <...>                // 其它终端工程
|- docs                         // 文档工程，所有团队共同维护，Asciidoc方案，后文会介绍
|   |- src
|   |   |- main
|   |   |   |- asciidoc
|   |   |   |   |- book.adoc    // 文档主目录
|   |   |   |   |- <...>        // 分模块建立不同的文档文件
|   |   |- resources            // 文档引用资源目录，如图片、CSS
|   |   |   |- images
|   |   |   |   |- <...>
|   |   |- pom.xml              // 文档工程使用Maven管理
|   |   |- .gitignore
|- env                          // 环境配置工程，以Spring Cloud Config为例，配置存放于Git
|   |- application.yml
|   |- <...>
|   |- .gitignore
|- pom.xml                      // 构建工程的POM
|- .gitmodules                  // Git子模块定义，所有工程都注册到此文件中
|- .gitignore
|- README.adoc                  // 构建工程使用说明
```

各个工程都对应于一个独立的Git库，Git库分四类：

**源码库（对应于sources目录下）** 存放所有的源代码，可细分成：全局父工程（sources/basics/parent），这是除终端外所有源码的父POM，全局公共工程（sources/basics/common）,这是除终端外所有源码都依赖的包，对于各服务组件还有全局公共服务工程（sources/basics/common-service）,它定义了包含公共启动文件、Domain、DTO等所有服务组件都需要的代码。服务组件指的就是一个个微服务（sources/services/...）,也是一个个独立的Git工程。所有终端代码也可被统一管理，要注意的是这里整体使用Maven，但IOS、Wechat都不是java工程，所以需要在这些工程中加一个只标记用于管理的POM文件（只有maven核心定义，不需要dependencies、build等）,如果我们的项目是服务平台类型，那么多半会需要对外提供SDK，所以需要有相应的SDK工程。

**文档库（对应于docs目录）** 推荐使用Asciidoc，使用[asciidoctor-maven-plugin](https://asciidoctor.org/docs/asciidoctor-maven-plugin/) 生成HTML、PDF等文档格式。

>📖 Asciidoc是一种标记文档格式，比Markdown语法更丰富，非常适合写复杂的文档，被Spring Cloud、HBase等众多开源框架所使用。

**环境配置库（对应于env目录）** 用于存放各环境的配置文件，将配置文件放到Git是比较好的选择，Spring Cloud、Vert.x都支持这一形式，后面我们会有独立章节探讨，统一的配置管理非常重要，可参见上一章节的相关介绍。

**构建库（对应于根目录）** 用对将各个独立的工程聚合管理。

>❓ 为什么文档库及配置库是整体项目独一份的?
>
>上文我说过自由是相对的，基础的规范要统一，这里的文档库是包含项目的概要设计、各服务的详细设计、发布规范、编码规范、技术选型规范、降级流程等，统一的文档库能让项目整体上更统一也更利于管理，配置库的统一可方便运维管理及配置依赖管理。

我们有了这么多独立的Git工程，看起来各工程都拆开了、独立了，但怎么有效的管理呢？这时我们就需要Git的Submodules了。上文提到的构建库正是用于这个目的。


>📖 Submodules是Git用于组织多个Git工程的一种方案，[详见此处](https://git-scm.com/book/en/v2/Git-Tools-Submodules)。

Submodules的核心是.gitmodules文件，它定义了不同Git工程的组织方式，示例如下：

```
[submodule "parent"]                        // 定义了parent工程
	path = sources/basics/parent            // 该工程在Build项目中的存放路径
	url = https://.../<group>/ parent.git   // 该工程的Git版本库地址
	branch = dev                            // 该工程对应的Git版本库分支
[submodule "common"]                        // 定义了common工程
	path = sources/basics/common
	url = https://.../<group>/common.git
	branch = dev
……
```

有了这份文件后我们就可以通过如下命令完成整体项目的检出：

`git clone --recursive <Build工程的Git地址> <project name>` ，--recursive参数会遍历此Git库中的Submodules定义并clone下各个子模块工程。

build.git工程包含了所有工程源码，多用于项目的整体管理，权限持有人为项目管理人员或总架构师，我们会为不同的团队设置不同的build工程，不同build工程包含不同的Submodules。如给短信团队的sms-build.git，它的.gitmodules如下：

```
[submodule "parent"]
	path = sources/basics/parent
	url = https://.../<group>/ parent.git
	branch = dev
[submodule "common"]
	path = sources/basics/common
	url = https://.../<group>/common.git
	branch = dev
[submodule "common-service"]
	path = sources/basics/common-service
	url = https://.../<group>/common-service.git
	branch = dev
[submodule "service-sms"]
	path = sources/services/service-sms
	url = https://.../<group>/service-sms.git
	branch = dev
```

我们看到，短信团队只要组装几个基础工程外加一个短信服务就可以了。

Git的Submodules对分支支持不是很友好，目前需要手工checkout到指定分支，我们可使用 `git submodule foreach` 来批量操作，如 `git submodule foreach 'git checkout master'` 。在子模块修改及多模块提交上可通过IDE以简化操作。以IDEA为例，如下图：

![](https://raw.githubusercontent.com/gudaoxuri/Microservices-Architecture/master/resources/images/ms-code-management-idea1.png)

IDEA可以感知到项目中的所有Git工程并询问是否加入到版本控制中，上图<Project>为我们的build工程，其它的为各个子模块。在我们修改任何被纳入IDEA管理的子模块时都可以一并被提交，提交时还可以选择哪些子模块是否提交：

![](https://raw.githubusercontent.com/gudaoxuri/Microservices-Architecture/master/resources/images/ms-code-management-idea2.png)

通过拆分各块到不同的版本库及使用Git的Submodules，我们可以很方便地管理众多的服务，非常推荐微服务团队采用，它所能带好的好处总结而言如下：

* **效率提升** 各团队不需要获取完整的项目代码，只要拉取自己团队所要关注的工程即可，开发中可以更聚焦，也加快了编译构建的速度
* **安全可控** 同样由于上面的原因，我们可以尽可能细粒度地控制代码的可见性，只给需要的人以权限
* **流程敏捷** 一般而言在发布之前我们都要做分支合并（参见不同的版本flow模型 ），发布分支只有特定人员才能合并，在不分工程的情况下大家试想下分支的合并场景， 我们需要会花很大的精力在Code Review及冲突解决上，在CI流程下如使用特定的分支用于测试那会导致非常频繁的触发自动构建，当我们拆分工程后一切都会变得简单，各团队负责人执行各自工程的分支合并，测试也会基于独立的分支进行

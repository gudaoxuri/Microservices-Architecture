# 全局ID策略

Id（Identification）是我们熟知的概念，用于标注对象身份，比如表主键、订单号、交易流水号、业务编码等，这些字段要确保全局唯一，在分布式环境下视场景的不同可以从极为简单到非常复杂，以订单号为例，不同的系统下订单号需求可能是随机的，可能是要求有顺序的，为了避免被竞争对手分析也可能要求是顺序与随机组合的（比如高位为日期，低位为随机数），不同的系统规模实现的手段可能是完全不同的，一些大型平台的Id生成服务会由成百上千个节点组成。总的来看全局Id策略的实现常见于以下几种：

数据库主键自增，这是最普通的方式，使用也最广，我们也可以利用一些数据库（如Oracle、Postgres）提供的Sequence特性，如果数据库不支持（如MySQL）也有一些变通的方案（请自行搜索），但在分库分表的情况下主键自增或Sequence会较为麻烦，一般采用两种方案，一是规划好库的个数确认id增长步长，比如分两张表，表一id从1开始 表二id从2开始,步长为2不会互相重复,缺点就是会有id的产生的顺序可能不连续,但是不会重复，另一个方案是使用类似[Flickr](http://code.flickr.net/2010/02/08/ticket-servers-distributed-unique-primary-keys-on-the-cheap/)的算法，Flickr基于MySQL，官方描述的核心算法很简单：

    ```
    # 创建一张Id生成表
    CREATE TABLE `Tickets64` (
      `id` bigint(20) unsigned NOT NULL auto_increment,
      `stub` char(1) NOT NULL default '', # Id的类型，如订单、用户等
      PRIMARY KEY  (`id`),
      UNIQUE KEY `stub` (`stub`)
    ) ENGINE=MyISAM # MyISAM性能高

    # 分表情况下获取Id
    REPLACE INTO Tickets64 (stub) VALUES ('a');  # REPLACE INFO 等同于 INSERT ON DUPLICATE KEY UPDATE
    SELECT LAST_INSERT_ID();  # 获取同一Connection中最后一次插入记录的自增值

    # 分库情况下获取Id,为避免跨库查询我们需要在各个库中都建立上述的Id生成表，然后为不同库下的Id生成表设置不同的起始值及步长，如有分个库则设置如下：

    库1的Tickets64，从1开始步长为2
    auto-increment-increment = 2
    auto-increment-offset = 1

    库2的Tickets64，从2开始步长为2
    auto-increment-increment = 2
    auto-increment-offset = 2
    ```

    Flickr不用像第一个方案一样要规划各表或库的数据量并且按时间排序时不会出现Id大范围地跳跃，是比较理想的基于数据库实现全局Id的方案，并且Flickr也对数据库高可用提供支持，比如我们将上述的库1视为主库，库2视为备库，在库1宕机启用库2时库2仍可提供有效的Id生成。但在分布式环境下基于数据库的方案场景比较局限，性能也有一定问题。

为解决上述问题，我们会比较自然地想到用[UUID](http://www.ietf.org/rfc/rfc4122.txt)，它不依赖于数据库，可非常简单地生成全球唯一的Id，这也是很常见的用法。UUID的问题在于它是乱序的，可读性差，常用数据库（如MySQL，B-Tree索引，数据存储在相邻的磁盘上，如果查询和写入的 Id 连续，可减少随机读写硬盘的几率）在插入、查询性能都明显差于自增Id，32位字符串（去掉中划线）占用空间大等。

乱序的问题可以参考有序GUID及Comb算法解决，Github上有相关的Java实现，字符串占用空间问题也可使用 `UUID.randomUUID().getMostSignificantBits() & Long.MAX_VALUE` 转成Long，但有一定重复风险，UUID.randomUUID()目前使用的是第四类UUID生成方案，它的重复概率是2^61，而转成Long为2^30[参见此处](https://stackoverflow.com/questions/325443/likelihood-of-collision-using-most-significant-bits-of-a-uuid-in-java)。

UUID及其变种的共同问题是它们都不是连续自增，一些场景下，比如报名序号，我们希望是自增的，我们可以考虑使用Redis的INCR这一原子自增操作来生成唯一Id，当然还有如Hazelcast、Ignite、Zookeeper、Consul等中间件都可实现类似功能，在大型高并发系统中要实现顺序ID基本都会基于一定的基础中间件实现一套大规模的ID生成服务。这一方案的问题在于都需要额外的服务，且生成Id都需要有一次请求操作（可部分引入微批次处理），在性能及复杂度上都会有一定影响。

如果我们需要不依赖数据库及中间件生成相对有序（非自增）、可读的全局Id，那么可以考虑使用[Snowflake](https://blog.twitter.com/engineering/en_us/a/2010/announcing-snowflake.html)及类似算法，Snowflake算法由Twitter开源，它的核心思想是用Long表示一个Id，一个Long型64位，最高位固定标识为正数，前41位为时间戳，到毫秒，中间10位为服务节点Id（前5位是数据中心Id、后5位是机器Id），最多支持1024个节点，最后12位是序号，支持一个服务节点在一毫秒内最多生成4096条记录，代码比较简单[见此处](https://github.com/twitter/snowflake/blob/snowflake-2010/src/main/scala/com/twitter/service/snowflake/IdWorker.scala)，当然也有其它语言及修改版本的实现，读者可自行到Github上检索。

Snowflake对时钟强依赖，调整机器时间会影响Id的唯一性，所以在运行时关闭NTP同步或在每次拿Id时记录当前时间，下次拿Id时判断时间是否小于上一次时间以进行一定的修正操作，Snowflake的另一个问题在于容器化部署时服务节点Id不好控制，容器节点会漂移且比较难通过IP及服务端口来确定唯一的节点，微服务与容器化是最佳的组合，所以在微服务下使用未经修改的Snowflake方案需要慎重。百度推出的[uid-generator](https://github.com/baidu/uid-generator/blob/master/README.zh_cn.md)正好可以解决这一问题，它基于Snowflake，将原本10位的服务节点Id扩展成22位，服务每次启动都向数据库（默认实现）获取一个不重复的服务节点Id，默认分配策略为用后即弃，最多支持420w次的机器启动。

在业务体量不多时全局Id非常容易实现，但在大体量下这需要每个架构师认真对待，一个优秀的全局Id方案要做到如下几点：

* 符合业务要求，位数、是否纯数字、随机还是顺序、是否允许不连续、是否需要基于一定要一规则等
* 高效，Id的获取会是非常频繁的操作，所以性能是考察的重点
* 稳定，做为核心的基础服务，稳定、高可用是必须关注的
* 简单，在满足上几条后我们在架构、使用层面都要尽可能地简单化

上文介绍了常用的策略，在实现设计中需要结合我们之前讨论缓存、幂等等方案灵活运用。


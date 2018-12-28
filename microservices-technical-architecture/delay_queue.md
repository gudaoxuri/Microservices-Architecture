# 延迟队列

上节介绍了顺序处理，我们实际场景中还会遇到诸如这些情况，比如对页面操作记录时要求操作事件是顺序的：Beforeload必须先于Unload，事件由同一个终端设备发送，通过设备ID Hash到同一个节点服务处理，这之中不存在时钟一致性问题，但由于事件发送是异步的，所以接收可能乱序，再比如在大数据系统中分析OAuth关系，OAuth表记录的是A应用的X用户与B应用的Y用户的关联（如果B应用没有对应的用户则Y用户为新增记录），但用户表、应用表和OAuth表都是分开采集的，即不能保证分析OAuth表时用户表对应的用户就一定已经存在。对于这类需求比较通用的解决方案是使用延迟队列。

顾名思义，延迟队列就是将处理按一定的要求延迟执行，针对上述需求可以在判断依赖记录未满足时延迟一段时间后再执行，另外延迟队列也可以处理诸如针对每个未付款的订单5分钟后发短信提醒、1天后关闭订单这种与记录实例相关的定时任务。

延迟队列的分单机与分布式，单机版本中可以使用Java自带的Timer或ScheduleExecutorService组件，使用定时任务的功能变相地实现延迟，当然Java也提供了DelayQueue专门用于处理延迟，这是单机下的首选。在分布式环境下实现有很多种方案：

* **轮询** 将任务写入到数据库、内存或其它存储介质中，启线程定期轮询是否到时间并执行相应的动作
* **Quartz** Quartz这知名的调度类库，基于此可实现分布式下的延迟调度
* **MQ** 一些MQ自带延迟消息，如阿里开源的RocketMQ，RabbitMQ通过[插件](https://github.com/rabbitmq/rabbitmq-delayed-message-exchange)及[Dead Letter Exchanges](http://www.rabbitmq.com/dlx.html)变相实现
* **Redis** Redis的Zset数据结构在延迟处理上使用的比较多
* **[TimerWheel](https://lwn.net/Articles/646950/)** 时间轮是一种高效的延迟设计方案，RocketMQ的内核正是基于此方案

那么如何选择呢？轮询的方案效率低实现简单，在并发不高的情况下可优先考虑，如果系统中已有定时任务处理，那么Quartz会是比较合适的选择，如果用了RocketMQ那么使用其延迟消息会是最佳方式，如果要自己设计一套独立的延迟队列服务，那么基于Redis的Zset或TimerWheel可以做为服务的内核使用。

下面是基于Redis Zset实现延迟队列的伪代码：

```scala
// --------------- 核心思想 ---------------
 hset 记录消息内容， zset 实现按到期时间排序的队列
 支持对同一个消息（kind与id相同）的延迟时间修改
// ----------------------------------------

// ----------- 延迟消息写入逻辑 -----------
// 保存消息内容，timerTaskReq.kind是消息类型，如订单到期、OAuth延迟处理等，timerTaskReq.id是消息的记录Id
redis.hset("delay:body:"+timerTaskReq.kind, timerTaskReq.id, toJsonString(timerTaskReq))
// 删除之前的延迟时间（如果存在的话）
redis.zrem("delay:queue:" + timerTaskReq.kind, timerTaskReq.id)
// 添加新的延迟时间，timerTaskReq.execMs为期望执行（到期）的时间，这里用这个时间做为评分
redis.zadd("delay:queue:" + timerTaskReq.kind, timerTaskReq.execMs, timerTaskReq.id)
// ----------------------------------------

// -------- 延迟消息获取及发送逻辑 --------
// kinds为所有的消息类型
kinds.map{
	kind ->
	// 获取过期的消息Id，即评分为0到当前时间戳
	var expireTaskIds = redis.zrangebyscore("delay:queue:" + kind, 0, currentTimeMs)
	// 获取对应的消息内容
	var expireTasks = redis.hmget("delay:body:" + kind, expireTaskIds)
	// 删除过期的消息Id
	redis.zremMany("delay:queue:" + kind, expireTaskIds)
	// 删除过期的消息内容
	redis.hdelMany("delay:body:" + kind, expireTaskIds)
	// 发送消息
	sendTask(expireTasks)
}
// ----------------------------------------
```

上面是自己实现一个延迟时间可修改的延迟队列内核的简化版本，是以Redis的zset结构为核心构建。

>❓ 是否可以使用Redis的Key过期通知 (https://redis.io/topics/notifications) 来实现延迟？
>
> 答案是不可以。
>
>如官方所言 "If no command targets the key constantly, and there are many keys with a TTL associated, there can be a significant delay between the time the key time to live drops to zero, and the time the expired event is generated.Basically expired events are generated when the Redis server deletes the key and not when the time to live theoretically reaches the value of zero."
>
>Redis只在过期键被删除的时候通知，而不是键的生存时间变为0的时候立马通知。过期键的删除是由一个后台任务执行，为不影响关键业务，后台任务被严格限制，默认为一秒执行10次，一次最多250ms，可通过hz参数调整，但当过期键比例很高时仍然会出现大量的通知的延迟。


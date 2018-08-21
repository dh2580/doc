Redis 知识点总结
===============
## 1、Redis应用场景
>1、会话缓存（`Session Cache`）
>
>2、后端数据缓存
>
>3、排行榜/计数器
>
>4、消息队列
>
>5、分布式锁

## 2、查看某个`key`的内部编码
> `object encoding xxxkey`

## 3、`Redis` 命令执行过程
> 一条命令从客户端到达服务端不会立刻被执行，所有的命令会进入一个队列中，然后被逐个执行

## 4、`Redis` 架构模型
> 采用纯内存访问模式（将所有数据均存放在内存中）、单线程模型、采用IO多路复用技术（epoll作为该实现）

## 5、单个Value的最大限制
> 1024M

## 6、单个字符串类型的值最大限制
> 512M
>
>来自`《Redis开发与运维》`一书

## 7、`String`类型的内部编码
>+ `int`：8个字节的长整形
>+ `embstr`：小于等于39个字节的字符串
>+ `raw`：大于39个字节的字符串

## 8、`Hash`类型的内部编码
>### 1、`ziplist`（压缩列表）
> 元素个数小于 `hash-max-ziplist-entries` （默认值512），并且所有值的大小都小于 `hash-max-ziplist-value` （默认值64字节），该方案采用更加紧凑的结构实现多个元素的连续存储，较为节省内存
>### 2、`hashtable`（哈希表）
> 无法满足 `ziplist` 存储条件时，采用该方案,因为数据较多的情况下，`ziplist`的读写效率会有所下降。`hashtable` 的读写复杂度为 `O(1)`

## 9、`List`类型最多可以存储多少个元素
> 2<sup>32</sup> - 1

## 10、`List`类型阻塞弹出操作
>### `blpop` 和 `brpop`
> 如果是`brpop`对多个键操作，则会从左至右遍历键，一旦有一个键能够弹出元素，则返回。`blpop`也是一样。
>
> 如果是多个客户端对同一个键执行`brpop`，那么最先执行`brpop`的客户端最先获取到弹出的值

## 11、`List`内部编码
>### 1、`ziplist`（压缩列表）
> 元素个数小于 `list-max-ziplist-entries`（默认值512个），同时每个元素的大小小于 `list-max-ziplist-value`（默认64个字节）
>### 2、`linkedlist`（链表）
> 不满足`ziplist`的使用条件时

## 12、`Set`内部编码
>### 1、`intset`（整数集合）
> 元素都是整数，并且元素个数小于`set-max-intset-entries`（默认值512个）
>### 2、`hashtable`（哈希表）
> 不满足`intset`的使用条件时

## 13、`ZSet`内部编码
>### 1、`ziplist`（压缩列表）
> 元素个数小于`zset-max-ziplist-entries`（默认值128个），同时每个元素的大小小于`zset-max-ziplist-value`（默认值128个字节）
>### 2、`skiplist`（跳跃表）
> 不满足`ziplist`的使用条件时

## 13、命令`zadd`的时间复杂度
> `O(log(n))`

## 14、命令`sadd`的时间复杂度
> `O(1)`

## 15、`persist`命令作用
> 可以删除任意类型键的过期时间。除了`persist`命令 之外，`set`命令也可以删除字符串类型`key`的过期时间

## 16、慢查询记录
>### 1、`slowlog-log-slower-than`配置
> 单位微妙，执行时间超过该配置的命令会被记录
>### 2、`slowlog-max-len`配置
> 慢查询日志最多存储的条数
>### 3、修改配置的方式
>+ 修改配置文件
>+ 采用`config`命令动态修改

    config set slowlog-log-slower-than 20000
    config set slowlog-max-len 1000
    config rewrite

>### 4、获取慢查询日志
> `slowlog get [n]`
>### 5、获取慢查询日志的长度
> `slowlog len`
>### 6、慢查询日志重置
> `slowlog reset`

## 17、`Pipeline`（流水线）工作机制
> 将一组`Redis`命令进行组装，通过一次`RTT`（`Round Trip Time`网络往返时间）传输给`Redis`,再将这组`Redis`命令的执行结果按照顺序返回给客户端。
>
>这么做的原因是因为`Redis`命令的真正执行时间通常在微妙级别，`Redis`的性能瓶颈在于网络传输上面

## 18、`Redis`事务
>### `Redis`支持简单的事务，但是不支持回滚功能
> `Redis`通过`multi`和`exec`来执行简单的事务，如下所示

    multi
    sadd class1 tom
    sadd class2 jack
    exec

> 如果在上述命令中，出现了命令错误，比如`sadd`写成了`saddd`，则会导致事务内整组命令不执行
>
>如果在上述命令中，出现了运行时错误，比如`sadd`写错了`zadd`，则不会导致事务内的其他命令不执行，即不支持回滚功能

## 19、`Lua`脚本的实现原理
> 简单来说，就是在`Redis`服务端执行`Lua`代码的时候，`Lua`代码将被当成一个命令去执行，并且直到`Redis`服务端执行完成，才会执行其他命令。

## 20、`Redis`之发布订阅
>1、 发布消息
>
>`publish channel message`
>
>2、订阅消息
>
>`subscribe channel [channel] ...`
>
>3、取消订阅
>
>`unsubscribe channel [channel] ...`
>
>4、按照模式订阅
>
>`psubscribe pattern [pattern] ...`
>
>5、按照模式取消订阅
>
>`punsubscribe pattern [pattern] ...`
>
>6、查看活跃的频道数量
>
>`pubsub channels [pattern] ...`
>
>7、查看频道订阅数
>
>`pubsub numsub [channel] ...`
>
>8、查看模式订阅数
>
>`pubsub numpat`

## 21、`Jedis`使用连接池的好处
>1、复用连接，减少创建/关闭连接带来的系统开销
>
>2、可以资源进行有效的控制，降低连接泄露的可能

## 22、`Redis`客户端管理
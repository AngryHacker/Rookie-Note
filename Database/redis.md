## Redis
### 数据库类型
KV 存储，非关系型（NoSQL）

### 存储量级
在内存存储，存储数量不多，配置中的 maxmemory 控制使用的物理内存。单例能存储 Key 2.5亿个(https://redis.io/topics/faq)，一个key或是value大小最大是512M。

容量评估：[Redis容量评估模型](https://cloud.tencent.com/developer/article/1004898)

超出使用内存：受配置中 maxmemory-policy 策略控制，默认是 noeviction（访问出错，不做淘汰）

### 内存模型
[Redis内存模型](https://www.cnblogs.com/kismetv/p/8654978.html)

### 线程模型
单线程，IO多路复用

### 读写速度


### 持久化机制
[Redis 持久化](https://www.cnblogs.com/kismetv/p/9137897.html)

### 锁机制
因为 redis 是单线程，对 redis 的数据读取不需要锁。但可以利用 redis 提供的锁机制做并发控制。
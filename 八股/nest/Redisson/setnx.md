SETNX 和 Redisson 都和 Redis 分布式锁有关，但层级不一样：

```
SETNX：Redis 的底层命令/加锁原语
Redisson：Java 里的 Redis 客户端 + 分布式工具库
```

可以理解成：

```
SETNX 是砖头
Redisson 是已经盖好的工具房
```

**1. SETNX 是什么**

SETNX 全称：

```
SET if Not eXists
```

意思是：

```
只有 key 不存在时，才设置成功
```

老写法：

```bash
SETNX lock:order:123 value
EXPIRE lock:order:123 10
```

但这个写法有问题。

如果：

```
SETNX 成功
还没来得及 EXPIRE
服务宕机
```

锁就没有过期时间，变成死锁。

所以现在推荐用 Redis 的 SET NX EX 原子命令：

```bash
SET lock:order:123 randomValue NX EX 10
```

含义：

```
NX：不存在才设置
EX 10：10 秒过期
randomValue：锁持有者标识
```

这一步是原子的。

**2. 用 SET NX EX 实现分布式锁**

加锁：

```bash
SET lock:order:123 uuid-1 NX EX 10
```

返回：

```
OK：拿锁成功
null：锁已存在，拿锁失败
```

释放锁不能直接：

```bash
DEL lock:order:123
```

因为可能误删别人的锁。

正确释放要用 Lua：

```lua
if redis.call("GET", KEYS[1]) == ARGV[1] then
  return redis.call("DEL", KEYS[1])
else
  return 0
end
```

意思是：

```
只有 value 等于我的 uuid，才删除
```

这就是手写 Redis 分布式锁的基础。

**3. SETNX 锁的基本问题**

自己用 SETNX/SET NX EX 写锁，要处理很多细节：

```
锁要有过期时间
释放锁要校验 value
释放要用 Lua 保证原子性
业务超时要不要续期
拿不到锁要不要等待
等待多久
是否可重入
是否公平
Redis 主从切换时怎么办
```

比如任务执行超过锁 TTL：

```
A 拿锁 10 秒
A 执行 20 秒
10 秒后锁过期
B 拿锁
A/B 同时执行
```

这就需要 watchdog 自动续期，或者重新设计任务幂等。

**4. Redisson 是什么**

Redisson 是 Java 生态里很常用的 Redis 客户端。

它不只是操作 Redis，还提供很多分布式工具：

```
分布式锁 RLock
公平锁
读写锁
信号量
闭锁 CountDownLatch
分布式集合
延迟队列
限流器
布隆过滤器
```

你在 Spring Boot 项目里常见：

```java
RLock lock = redissonClient.getLock("lock:order:123");

lock.lock();

try {
    // 业务逻辑
} finally {
    lock.unlock();
}
```

它底层其实也是用 Redis 命令 + Lua 脚本实现的，只是帮你封装好了很多细节。

**5. Redisson 分布式锁示例**

```java
RLock lock = redissonClient.getLock("lock:order:" + orderId);

boolean locked = lock.tryLock(5, 30, TimeUnit.SECONDS);

if (!locked) {
    throw new RuntimeException("请稍后再试");
}

try {
    // 处理订单
} finally {
    lock.unlock();
}
```

这里：

```
tryLock(5, 30, TimeUnit.SECONDS)
```

含义大致是：

```
最多等 5 秒获取锁
获取成功后锁 30 秒自动释放
```

如果业务执行完：

```java
lock.unlock();
```

释放锁。

**6. Redisson Watchdog**

Redisson 的一个重要能力是 watchdog。

如果你这样写：

```java
lock.lock();
```

没有指定锁过期时间，Redisson 会启动 watchdog 自动续期。

默认逻辑类似：

```
锁过期时间 30 秒
每 10 秒续期一次
只要业务线程还活着，就持续续期
unlock 后停止续期
```

这样避免：

```
业务还没执行完，锁先过期
```

但如果你这样写：

```java
lock.lock(10, TimeUnit.SECONDS);
```

指定了 leaseTime，通常就不会无限续期，到 10 秒自动释放。

**7. Redisson 可重入锁**

Redisson 的 RLock 是可重入的，类似 Java 的 ReentrantLock。

同一个线程可以重复加同一把锁：

```java
lock.lock();
lock.lock();

try {
    // 同一个线程重入
} finally {
    lock.unlock();
    lock.unlock();
}
```

内部会记录重入次数。

自己用 SETNX 写锁，默认没有可重入能力。要实现就复杂很多。

**8. Redisson 和 SETNX 的关系**

可以这样理解：

```
SETNX / SET NX EX：
Redis 原生命令，用来实现锁的基础能力。

Redisson：
基于 Redis 命令和 Lua 脚本封装出的高级分布式锁工具。
```

对比：

```
SETNX:
  轻量
  灵活
  要自己处理细节
  容易写错

Redisson:
  封装成熟
  支持 watchdog
  支持可重入
  支持公平锁/读写锁等
  适合 Java/Spring Boot 项目
```

**9. 什么时候用 SET NX EX**

适合：

```
简单互斥
锁逻辑很短
你能接受自己处理释放 Lua
Node/NestJS 项目
不想引入复杂库
```

例如：

```
防止同一用户重复提交
热点缓存重建互斥
短时间任务互斥
```

但你要严格做到：

```
SET key value NX EX ttl
Lua 校验 value 删除
TTL 合理
业务幂等
```

**10. 什么时候用 Redisson**

适合：

```
Java/Spring Boot 项目
锁场景较多
需要可重入锁
需要 watchdog 自动续期
需要公平锁/读写锁
需要更成熟的分布式工具
```

比如：

`订单防重复处理 库存扣减保护 定时任务单实例执行 复杂业务互斥`

**11. 一个重要提醒**

不管是 SETNX 还是 Redisson，分布式锁都不是万能的。

高并发扣库存/秒杀里，不一定首选分布式锁。

更常见的是：

```
Redis 原子扣库存
MQ 异步下单
数据库唯一约束保证一人一单
幂等消费
最终一致性
```

分布式锁适合互斥控制，但如果每个请求都抢同一把锁，会严重降低吞吐。

**一句话总结**

```
SETNX 是 Redis 实现分布式锁的底层思路；
Redisson 是 Java 生态里基于 Redis 封装好的分布式锁工具，帮你处理可重入、watchdog、Lua 解锁等复杂细节。
```
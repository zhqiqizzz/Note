**Redisson** 是 Java 生态里最常用的 Redis 高级客户端之一，常配合 **Spring Boot** 使用。

你可以把它理解成：

```
Redis 原生命令客户端：Lettuce / Jedis
Redis 高级封装客户端：Redisson
```

Redisson 不只是 get/set，它把 Redis 封装成很多 Java 分布式对象和工具。

**Redisson 常用能力**

```
分布式锁 RLock
可重入锁
公平锁
读写锁
信号量
限流器
延迟队列
分布式集合 Map/List/Set
分布式 AtomicLong
布隆过滤器
Topic 发布订阅
```

最常用的是：**分布式锁**。

**为什么用 Redisson 做锁**

手写 Redis 锁要考虑：

```
SET NX EX
唯一 value
Lua 原子释放
锁超时
业务没执行完锁过期
自动续期
可重入
异常释放
```

Redisson 帮你封装了很多细节，尤其是 **watchdog 自动续期**。

**Redisson 分布式锁示例**

```java
RLock lock = redissonClient.getLock("lock:order:123");

try {
    boolean locked = lock.tryLock(3, 30, TimeUnit.SECONDS);

    if (!locked) {
        throw new RuntimeException("获取锁失败");
    }

    // 执行业务逻辑
    orderService.cancelOrder(123L);

} finally {
    if (lock.isHeldByCurrentThread()) {
        lock.unlock();
    }
}
```

这里：

```
tryLock(3, 30, TimeUnit.SECONDS)

3：最多等 3 秒拿锁
30：拿到锁后 30 秒自动释放
```

如果你用：

```java
lock.lock();
```

Redisson 默认会启用 watchdog 自动续期。  
也就是说，只要当前服务还活着，它会自动延长锁过期时间，避免业务没执行完锁就过期。

**Spring Boot 集成**

依赖：

```xml
<dependency>
    <groupId>org.redisson</groupId>
    <artifactId>redisson-spring-boot-starter</artifactId>
    <version>3.x.x</version>
</dependency>
```

配置：

```yaml
spring:
  redis:
    host: localhost
    port: 6379
```

或者单独配置 Redisson：

```yaml
redisson:
  config: |
    singleServerConfig:
      address: "redis://127.0.0.1:6379"
```

注入：

```java
@Resource
private RedissonClient redissonClient;
```

**典型场景**

```
防止重复下单
防止重复支付回调
同一 Agent run 只能一个 worker 执行
热点缓存重建互斥
定时任务多实例只执行一次
库存扣减保护
```

**注意点**

Redisson 锁不是万能的。

要注意：

```
锁粒度不要太大
锁内业务尽量短
finally 里释放锁
释放前判断 isHeldByCurrentThread
高并发扣库存不一定靠锁，常用 Redis 原子扣减 + MQ
核心金融场景还要数据库唯一约束/事务兜底
```

一句话总结：

```
Redisson 是 Java/Spring 里用 Redis 做分布式锁、限流、队列等高级能力的常用客户端，最核心价值是把 Redis 分布式锁的复杂细节封装好了。
```
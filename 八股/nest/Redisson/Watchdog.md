Redisson 的**看门狗 Watchdog**，核心作用是：

```
当前线程还在持有锁时，自动给 Redis 锁续期，防止业务没执行完锁就过期。
```

它解决的是这个问题：

```
锁默认 30 秒过期
业务执行了 60 秒
如果没人续期，30 秒后锁自动释放
其他线程拿到锁
两个线程同时执行业务，出并发问题
```

**1. 没有看门狗会怎样**

比如你手写 Redis 锁：

```
SET lock:order:123 value NX EX 30
```

表示锁 30 秒后自动过期。

如果业务 10 秒完成，没问题。

但如果业务因为网络、数据库慢、模型调用慢，执行了 60 秒：

```
第 0 秒：线程 A 拿到锁，TTL=30s
第 30 秒：锁过期，被 Redis 删除
第 31 秒：线程 B 拿到锁
第 31-60 秒：A 和 B 同时执行业务
```

这就破坏了锁的互斥性。

**2. Watchdog 怎么解决**

Redisson 如果你这样加锁：

```
lock.lock();
```

没有指定锁的租约时间 leaseTime，就会启动看门狗。

默认逻辑：

```
锁默认过期时间：30 秒
每隔 10 秒左右检查一次
如果当前线程仍持有锁
就把锁 TTL 重新续到 30 秒
```

时间线：

```
第 0 秒：A 获取锁，TTL=30s
第 10 秒：看门狗续期，TTL 回到 30s
第 20 秒：看门狗续期，TTL 回到 30s
第 30 秒：看门狗续期，TTL 回到 30s
...
第 60 秒：业务完成，A 主动 unlock
```

这样业务执行多久，锁就能一直被续住。

**3. 默认续期时间**

Redisson 默认锁过期时间通常是：

```
30 秒
```

配置项是：

```
lockWatchdogTimeout
```

可以配置：

```java
Config config = new Config();
config.setLockWatchdogTimeout(30000);
```

单位是毫秒。

看门狗通常按这个时间的三分之一间隔续期：

```
30 秒超时
大约每 10 秒续期一次
```

**4. 什么时候会启用 Watchdog**

会启用：

```java
lock.lock()
```

或者：

```java
lock.tryLock(waitTime, TimeUnit.SECONDS);
```

也就是你没有指定明确的 leaseTime。

不会启用或不依赖 watchdog：

```java
lock.lock(10, TimeUnit.SECONDS);
```

```java
lock.tryLock(3, 10, TimeUnit.SECONDS);
```

因为你已经明确说：

```
这把锁 10 秒后自动释放
```

Redisson 就按你指定的时间处理。

所以要记住：

```
30 秒超时
大约每 10 秒续期一次
```

**5. 为什么指定 leaseTime 反而可能危险**

比如：

```java
lock.tryLock(3, 10, TimeUnit.SECONDS);
```

含义：

```
最多等 3 秒拿锁
拿到后锁 10 秒自动释放
```

如果业务执行 20 秒：

```
第 0 秒：A 拿锁
第 10 秒：锁自动释放
第 11 秒：B 拿锁
第 11-20 秒：A 和 B 并发执行
```

所以指定 leaseTime 时，一定要保证：

```
业务执行时间 < leaseTime
```

否则就要考虑不用 leaseTime，让 watchdog 自动续期，或者把业务拆成可幂等任务。

**6. Watchdog 什么时候停止续期**

看门狗会在这些情况下停止：

```
业务执行完，调用 unlock
持锁线程结束
Redisson 客户端关闭
应用宕机
网络断开，客户端无法续期
```

如果应用宕机：

```
看门狗停止续期
Redis 锁最多等到 TTL 到期
自动释放
```

这就是为什么锁仍然必须有过期时间。

**7. Watchdog 续期的前提**

看门狗续期前会确认：

```
当前锁仍然存在
当前锁仍然属于这个线程
```

Redisson 锁底层是 Hash，大概像：

```
key: lock:order:123
field: clientId:threadId
value: 重入次数
```

续期时会判断：

```
这个 Hash 里是否还有当前 clientId:threadId
```

如果有，才续期。

如果锁已经被释放或不属于当前线程，就不续期。

**8. Watchdog 的价值**

它主要解决：

```
业务耗时不确定
不能准确设置锁过期时间
又不希望锁提前过期
```

例如：

```
调用第三方接口
处理大文件
生成报表
执行 Agent 长任务
复杂数据库事务
```

这些场景业务时间可能波动很大，看门狗能减少锁提前释放风险。

**9. Watchdog 不能解决所有问题**

看门狗不是万能的。

它不能解决：

```
业务线程卡死但进程还活着
网络分区导致续期异常
Redis 主从切换导致锁状态问题
锁粒度设计错误
业务本身不幂等
死锁式业务设计
```

比如线程一直卡住没释放锁，但 Redisson 还认为线程持锁，可能持续续期，导致其他请求长期拿不到锁。

所以生产里还要：

```
控制锁粒度
设置业务超时
做好幂等
避免长时间持锁
必要时用任务状态表辅助
```

**10. 推荐使用方式**

业务耗时短且可预估：

```java
boolean locked = lock.tryLock(3, 10, TimeUnit.SECONDS);
```

业务耗时不确定：

```java
boolean locked = lock.tryLock(3, TimeUnit.SECONDS);
```

或者：

```java
lock.lock();
```

然后：

```java
try {
    // business
} finally {
    if (lock.isHeldByCurrentThread()) {
        lock.unlock();
    }
}
```

更推荐有等待上限：

```java
boolean locked = lock.tryLock(3, TimeUnit.SECONDS);

if (!locked) {
    throw new RuntimeException("获取锁失败");
}
```

**一句话总结**

Redisson 看门狗就是：

```
在没有指定 leaseTime 的情况下，自动周期性延长锁的过期时间，确保业务没执行完时锁不会提前释放。
```

但它只能保护“锁不提前过期”，不能替代：

```
业务幂等
超时控制
合理锁粒度
异常处理
```
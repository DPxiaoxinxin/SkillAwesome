# Redis分布式锁

## 概述



本文尽量不采用具体语言，尽量用Redis命令来阐述。

### 使用的Redis命令

#### SET

``` redis
# 给key设置value，如key已存在，则覆盖Value。设置成功返回OK，失败返回NULL
SET key value
# 该命令含有4个关键参数，不同类型参数可以互相组合
# EX 过期时间，单位：秒; PX 过期时间，单位：毫秒
SET key value EX 10
# NX 只有在key不存在时设置; XX 只在key存在时设置
SET key value NX
```

#### EVAL

``` redis
# 使用lua解释器执行lua脚本，该操作是原子性的。
# 参数1是lua脚本
# 参数2是key的数量，已经对应key的值，传递进lua脚本的KEYS全局变量
# 参数3是所有传进lua脚本ARGV全局变量的值
> eval "return {KEYS[1],KEYS[2],ARGV[1],ARGV[2]}" 2 key1 key2 first second
1) "key1"
2) "key2"
3) "first"
4) "second"
```



## 分布式锁条件

### 互斥性

在任意一个时刻，只有一个客户端能持有锁

### 无死锁

即使有一个客户端在持有锁的期间崩溃而没有主动解锁，也能保证后续其它客户端能加锁

### 容错性

只要大部分的Redis节点都在运行，客户端就可以加锁和解锁

### 无误删

加锁和解锁必须是同一个客户端，客户端自己不能把别人的锁删了



## 方案

### 版本1：解决互斥性

#### 方法

` SET key value NX`

#### 问题

假设有客户端A，A获取到锁后挂了，该锁不会释放，其他客户端永远获取不到锁



### 版本2：解决死锁--设置过期时间

#### 方法

` SET key value NX time`

#### 问题

* 过期时间如何保证大于业务执行时间？
  * 假如有客户端AB，A获取锁后设置超时时间10s，但是A阻塞了20s，期间锁释放后B获得了同一个锁，对该锁执行了相同的业务逻辑
* 如何保证锁不会被误删？
  * 假如有客户端AB，A获取锁后设置超时时间10s，但是A阻塞了20s，期间锁释放后B获得了同一个锁，20s后A恢复，把B持有的锁删除

### 版本3：解决误删--锁设置客户端ID+原子性释放锁

#### 方法

``` redis
# 获取锁，value设置为全局唯一，比如说UUID随机值
SET key value NX time
# 释放锁
script = "if redis.call('get', KEYS[1]) == ARGV[1] then " +
                                "return redis.call('del', KEYS[1]) " +
                                "else " +
                                "return 0 " +
                                "end";
eval script 1 key value
```

### 版本4：确保过期时间大于业务执行时间

#### 思路

1. 该场景遇见情况很少，如果不影响一致性，可忽略，偶尔重复执行也没关系。
2. 给持有锁的线程增加守护线程，定时刷新过期时间，主线程结束，守护线程也结束。但可能导致锁迟迟不得释放，影响业务。

### 版本5：解决锁的重入性

#### 思路

* 存储锁的重入次数，以及分布式环境下唯一的线程标识

### 版本6：解决容错性

#### 思路

* 如果要保证强一致性，建议采用Zookeeper做分布式锁
* 参考RedLock算法：
  * 获取当前Unix时间，以毫秒为单位。
  * 依次尝试从N个实例，使用相同的key和随机值获取锁。
    * 当向Redis设置锁时,客户端应该设置一个网络连接和响应超时时间，这个超时时间应该小于锁的失效时间。例如你的锁自动失效时间为10秒，则超时时间应该在5-50毫秒之间。这样可以避免服务器端Redis已经挂掉的情况下，客户端还在死死地等待响应结果。如果服务器端没有在规定时间内响应，客户端应该尽快尝试另外一个Redis实例。
  * 客户端使用当前时间减去开始获取锁时间（步骤1记录的时间）就得到获取锁使用的时间。当且仅当从大多数（这里是3个节点）的Redis节点都取到锁，并且使用的时间小于锁失效时间时，锁才算获取成功。
  * 如果取到了锁，key的真正有效时间等于有效时间减去获取锁所使用的时间（步骤3计算的结果）。
  * 如果因为某些原因，获取锁失败（*没有*在至少N/2+1个Redis实例取到锁或者取锁时间已经超过了有效时间），客户端应该在所有的Redis实例上进行解锁（即便某些Redis实例根本就没有加锁成功）。
  * 解锁时向所有的Redis实例发送释放锁命令即可，不用关心之前有没有从Redis实例成功获取到锁.



## 总结

* 生成环境建议上成熟的分布式锁框架，如redis版本的`Redisson`，或者基于zookeeper的分布式框架
* 如果对一致性等要求没那么苛刻，可考虑按本文思路自行实现。
* 网上有些文章基于redis的分布式锁略复杂，使用了setnx和getset命令来解决死锁问题，可能是之前redis版本的set命令没有nx选项。这些不建议采用。



## 代码示例

``` java
public class RedisLock{

    private Jedis jedis;
    private String lockKey;
    protected String lockValue;

    class LockConstants {
        public static final String OK = "OK";

        public static final String NOT_EXIST = "NX";
        public static final String EXIST = "XX";

        public static final String SECONDS = "EX";
        public static final String MILLISECONDS = "PX";

        private LockConstants() {}
    }

    public RedisLock(Jedis jedis, String lockKey) {
        this(jedis, lockKey,UUID.randomUUID().toString()+Thread.currentThread().getId());
    }

    public RedisLock(Jedis jedis, String lockKey, String lockValue) {
        this.jedis = jedis;
        this.lockKey = lockKey;
        this.lockValue = lockValue;
    }

    public void lock() {
        while(true){
            String result = jedis.set(lockKey, lockValue, LockConstants.NOT_EXIST,LockConstants.SECONDS,30);
            if(LockConstants.OK.equals(result)){
                System.out.println(Thread.currentThread().getId()+"加锁成功!");
                break;
            }
        }
    }

    public void unlock() {
        // 使用lua脚本进行原子删除操作
        String checkAndDelScript = "if redis.call('get', KEYS[1]) == ARGV[1] then " +
                "return redis.call('del', KEYS[1]) " +
                "else " +
                "return 0 " +
                "end";
        jedis.eval(checkAndDelScript, 1, lockKey, lockValue);
    }
}

```

## 参考

[redis系列：分布式锁](https://juejin.im/post/5b737b9b518825613d3894f4)

[Redis 分布式锁的正确实现方式（ Java 版 ）](http://www.importnew.com/27477.html?replytocom=702052#respond)
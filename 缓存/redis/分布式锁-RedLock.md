[TOC]

------------------todo

<https://gitbook.cn/books/5b401d8d21cd1904a7114ab6/index.html>

# CAP

在引出RedLock之前，先介绍一下分布式系统中CAP理论：

C(Consistency)：一致性，在同一时间点，所有节点的数据都是完全一致的。

A(Availability)：可用性，应该能够在正常时间内对请求进行响应。

P(Partition-tolerance)：分区容忍性，在分布式环境中，多个节点组成的网络应该是互相连通的，当由于网络故障等原因造成网络分区，要求仍然能够对外提供服务。

CAP理论告诉我们，任何分布式系统只能满足三个中的两个，不可能全部都满足。

参考[分布式系统那点事](https://blog.csdn.net/stone_yw/article/details/87926450)

# redis分布式锁存在的问题

网上对于采用redis实现分布式锁有很多种方案，比较完善的方案应该是用setNx + lua进行实现。简单实现如下：

```lua
- java代码-加锁,相当于setnx lock_key_name unique_value
set lock_key_name unique_value NX PX 5000;
- lua脚本-解锁，原子性操作
if redis.call("get", KEYS[1] == ARGV[1]) then
	return redis.call("del", KEYS[1])
else
    return 0
end
```

<font color="00fffff">注意：</font>

1. value需要具有唯一性，可以采用时间戳、uuid或者自增id实现；
2. 客户端在解锁时，需要比较本地内存中的value和redis中的value是否一致，防止误解锁;（case：clientA获取锁lock1，由于clientA执行的时间比较久，导致key=lock1已经过期，redis实例会移除该key；clientB获取相同的锁lock1，clientB正在占有锁并执行业务，此时clientA业务已经执行完毕，准备释放锁；如果没有比较value的逻辑，那么clientA会把clientB持有的锁释放掉，这个显然不行的，由于value值不同，那么clientA释放锁的时候只会释放自己加的锁，不会误释放别的客户端加的锁）

在分布式系统中，为了避免单点故障，提高可靠性，redis都会采用主从架构，当主节点挂了后，从节点会作为主继续提供服务。该种方案能够满足大多数的业务场景，但是对于要求强一致性的场景如交易，该种方案还是有漏洞的，原因如下：

redis主从架构采用的是异步复制，当master节点拿到了锁，但是锁还未同步到slave节点，此时master节点挂了，发生故障转移，slave节点被选举为master节点，丢失了锁。这样其他线程就能够获取到该锁，显然是有问题的。

因此，上述基于redis实现的分布式锁只是满足了AP，并没有满足C。

# RedLock

正是因为上述redis分布式锁存在的一致性问题，redis作者提出了一个更加高级的基于redis实现的分布式锁——RedLock。原文可参考[ Distributed locks with Redis](https://redis.io/topics/distlock)

## RedLock是什么？

RedLock是基于redis实现的分布式锁，它能够保证以下特性：

1. 互斥性：在任何时候，只能有一个客户端能够持有锁；
2. 避免死锁：当客户端拿到锁后，即使发生了网络分区或者客户端宕机，也不会发生死锁；（利用key的存活时间）
3. 容错性：只要多数节点的redis实例正常运行，就能够对外提供服务，加锁或者释放锁；

而非redLock是无法满足互斥性的，上面已经阐述过了原因。

## RedLock算法

假设有N个redis的master节点，这些节点是相互独立的（不需要主从或者其他协调的系统）。<font color="ffff">N推荐为奇数~</font>

客户端在获取锁时，需要做以下操作：

1. 获取当前时间戳，以微妙为单为。
2. 使用相同的lockName和lockValue，尝试从N个节点获取锁。（在获取锁时，要求等待获取锁的时间远小于锁的释放时间，如锁的lease_time为10s，那么wait_time应该为5-50毫秒；避免因为redis实例挂掉，客户端需要等待更长的时间才能返回，即需要让客户端能够fast_fail；如果一个redis实例不可用，那么需要继续从下个redis实例获取锁）
3. 当从N个节点获取锁结束后，如果客户端能够从多数节点(N/2 + 1)中成功获取锁，且获取锁的时间小于失效时间，那么可认为，客户端成功获得了锁。（获取锁的时间=当前时间戳 - 步骤1的时间戳）
4. 客户端成功获得锁后，那么锁的实际有效时间 = 设置锁的有效时间 - 获取锁的时间。
5. 客户端获取锁失败后，N个节点的redis实例都会释放锁，即使未能加锁成功。

为什么N推荐为奇数呢？
原因1：本着最大容错的情况下，占用服务资源最少的原则，2N+1和2N+2的容灾能力是一样的，所以采用2N+1；比如，5台服务器允许2台宕机，容错性为2，6台服务器也只能允许2台宕机，容错性也是2，因为要求超过半数节点存活才OK。

原因2：假设有6个redis节点，client1和client2同时向redis实例获取同一个锁资源，那么可能发生的结果是——client1获得了3把锁，client2获得了3把锁，由于都没有超过半数，那么client1和client2获取锁都失败，对于奇数节点是不会存在这个问题。

## 失败时重试

当客户端无法获取到锁时，应该随机延时后进行重试，防止多个客户端在同一时间抢夺同一资源的锁（会导致脑裂，最终都不能获取到锁）。客户端获得超过半数节点的锁花费的时间越短，那么脑裂的概率就越低。所以，理想的情况下，客户端最好能够同时（并发）向所有redis发出set命令。

当客户端从多数节点获取锁失败时，应该尽快释放已经成功获取的锁，这样其他客户端不需要等待锁过期后再获取。（如果存在网络分区，客户端已经无法和redis进行通信，那么此时只能等待锁过期后自动释放）

<font color="ffff">不明白为什么会发生脑裂？？？</font>

## 释放锁

向所有redis实例发送释放锁命令即可，不需要关心redis实例有没有成功上锁。

<font color="ffff">redisson在加锁的时候，key=lockName, value=uuid + threadID, 采用set结构存储，并包含了上锁的次数 （支持可重入）；解锁的时候通过hexists判断key和value是否存在，存在则解锁；这里不会出现误解锁</font>

## 性能、 崩溃恢复和redis同步

如何提升分布式锁的性能？以每分钟执行多少次acquire/release操作作为性能指标，一方面通过增加redis实例可用降低响应延迟，另一方面，使用非阻塞模型，一次发送所有的命令，然后异步读取响应结果，这里假设客户端和redis之间的RTT差不多。

如果redis没用使用备份，redis重启后，那么会丢失锁，导致多个客户端都能获取到锁。通过AOF持久化可以缓解这个问题。redis key过期是unix时间戳，即便是redis重启，那么时间依然是前进的。但是，如果是断电呢？redis在启动后，可能就会丢失这个key（在写入或者还未写入磁盘时断电了，取决于fsync的配置），如果采用fsync=always，那么会极大影响性能。如何解决这个问题呢？可以让redis节点重启后，在一个TTL时间段内，对客户端不可用即可。

## 针对redlock的争议

[参考链接](http://martin.kleppmann.com/2016/02/08/how-to-do-distributed-locking.html

存在的问题:http://zhangtielei.com/posts/blog-redlock-reasoning.html
比如现在5个客户端，有三个获取成功，另外两个没有获取成功，此时有一个获取成功的宕机了，那么其他客户端依然可能成功获取锁。===解决方案：延迟重启，挂了的机器等锁过期后再重启。

redlock的前提是不要采用主从！！！直接多个redis单机就行了。

# Redisson

redisson是在redis基础上实现的一套开源解决方案，不仅提供了一系列的分布式的java常用对象，还提供了许多分布式服务，宗旨是促进使用者对redis的关注分离，更多的关注业务逻辑的处理上。

redisson也对redlock做了一套实现，详细如下：

## 使用案例

```java
public static void main() {

        Config config1 = new Config();
        config1.useSingleServer().setAddress("redis://xxxx1:xxx1")
                .setPassword("xxxx1")
                .setDatabase(0);
        RedissonClient redissonClient1 = Redisson.create(config1);
        Config config2 = new Config();
        config2.useSingleServer()
                .setAddress("redis://xxxx2:xxx2")
                .setPassword("xxxx2")
                .setDatabase(0);

        RedissonClient redissonClient2 = Redisson.create(config2);

        Config config3 = new Config();
        config3.useSingleServer().
                setAddress("redis://xxxx3:xxx3")
                .setPassword("xxxx3")
                .setDatabase(0);

        RedissonClient redissonClient3 = Redisson.create(config3);

        String lockName = "redlock-test";
        RLock lock1 = redissonClient1.getLock(lockName);
        RLock lock2 = redissonClient2.getLock(lockName);
        RLock lock3 = redissonClient3.getLock(lockName);

        RedissonRedLock redLock = new RedissonRedLock(lock1, lock2, lock3);
        boolean isLock;
        try {
            isLock = redLock.tryLock(500, 30000, TimeUnit.MILLISECONDS);
            System.out.println("isLock = " + isLock);
            if (isLock) {
                // lock success, do something;
                Thread.sleep(30000);
            }
        } catch (Exception e) {

        } finally {
            // 无论如何, 最后都要解锁
            redLock.unlock();
            System.out.println("unlock success");
        }
    }
```

## 源码

tryLock()：redisson对redlock的实现方式基本和上述描述的类似，有一点区别在于，redisson在获取锁成功后，会对key的失效时间重新。

```java
public boolean tryLock(long waitTime, long leaseTime, TimeUnit unit) throws InterruptedException {
        long newLeaseTime = -1;
        if (leaseTime != -1) {
            newLeaseTime = unit.toMillis(waitTime)*2;
        }
        
        long time = System.currentTimeMillis();
        long remainTime = -1;
        if (waitTime != -1) {
            remainTime = unit.toMillis(waitTime);
        }
        long lockWaitTime = calcLockWaitTime(remainTime);
        
        int failedLocksLimit = failedLocksLimit();
        List<RLock> acquiredLocks = new ArrayList<RLock>(locks.size());
        for (ListIterator<RLock> iterator = locks.listIterator(); iterator.hasNext();) {
            RLock lock = iterator.next();
            boolean lockAcquired;
            try {
                if (waitTime == -1 && leaseTime == -1) {
                    lockAcquired = lock.tryLock();
                } else {
                    long awaitTime = Math.min(lockWaitTime, remainTime);
                    lockAcquired = lock.tryLock(awaitTime, newLeaseTime, TimeUnit.MILLISECONDS);
                }
            } catch (RedisResponseTimeoutException e) {
                unlockInner(Arrays.asList(lock));
                lockAcquired = false;
            } catch (Exception e) {
                lockAcquired = false;
            }
            
            if (lockAcquired) {
                acquiredLocks.add(lock);
            } else {
                if (locks.size() - acquiredLocks.size() == failedLocksLimit()) {
                    break;
                }

                if (failedLocksLimit == 0) {
                    unlockInner(acquiredLocks);
                    if (waitTime == -1 && leaseTime == -1) {
                        return false;
                    }
                    failedLocksLimit = failedLocksLimit();
                    acquiredLocks.clear();
                    // reset iterator
                    while (iterator.hasPrevious()) {
                        iterator.previous();
                    }
                } else {
                    failedLocksLimit--;
                }
            }
            
            if (remainTime != -1) {
                remainTime -= (System.currentTimeMillis() - time);
                time = System.currentTimeMillis();
                if (remainTime <= 0) {
                    unlockInner(acquiredLocks);
                    return false;
                }
            }
        }

        if (leaseTime != -1) {
            List<RFuture<Boolean>> futures = new ArrayList<RFuture<Boolean>>(acquiredLocks.size());
            for (RLock rLock : acquiredLocks) {
                RFuture<Boolean> future = rLock.expireAsync(unit.toMillis(leaseTime), TimeUnit.MILLISECONDS);
                futures.add(future);
            }
            
            for (RFuture<Boolean> rFuture : futures) {
                rFuture.syncUninterruptibly();
            }
        }
        
        return true;
    }
```


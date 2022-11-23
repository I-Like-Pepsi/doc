# Redis实现分布式锁

## 简介

在Java单机应用中对于多线程资源安全问题我们通常使用JDK中提供的锁即可完成，但是在分布式环境中JDK中提供的锁很显然无法满足业务要求。通常情况下分布式锁的实现方案可以通过Redis、Zookeeper来实现，本文这里只讨论使用Redis来实现分布式锁。

## 如何实现

不管是分布式环境中的锁还是单机应用中的锁的两个主要操作即使Lock（获取锁）和UnLock（释放锁），而对应到Redis中就是向Redis中写入一个数据和删除一个数据。

- 加锁命令在Redis中通常使用```SETNX key value```来实现。当键不存在的时，对键进行设置操作并返回成功，否则返回失败。这分别代表了获取锁成功和获取锁失败，因为只有一个线程能操作成功。

- 解锁命令在Redis中通常使用```DEL key```来实现，通过删除键值释放锁，以便于后来的线程获取锁。

- 锁操作通常使用```EXPIRE key timeout```设置超时时间，以保证锁被客户端释放资源一直被占用。

## 问题

上面只是简单的说了一下如何实现，然而事实上并没有这么简单。

### SETNX和EXPIRE非原子性

如果我们使用**SETNX**成功了，准备使用**EXPIRE**给键设置超时时间，然后因为网络问题或者其他问题**EXPIRE**操作失败，这将导致锁没有超时时间而资源一直被占用从而发生**死锁**现象。这个问题的核心是因为两个操作非原子操作，要解决这个问题就是让其变成原子操作，通常有两种方式来解决。

#### SET增加NX和EX参数

在**2.6.12**版本，Redis的**SET**增加了**NX**和**EX**参数，从而实现了设置key不存则成功和设置key超时时间，并且该操作是一个原子操作。通常命令如下：

```shell
SET resource-name anystring NX EX max-lock-time
```

但是该方式有一个缺点，它要求Redis的版本必须大于等于**2.6.12**，如果Redis版本比较老很可能无法实现。

#### Lua脚本

上面的方式对Redis版本有要求，但是我们可以通过Lua脚本来实现该保证两个操作的原子性。通常脚本如下：

```lua
if (redis.call('setnx', KEYS[1], ARGV[1]) < 1)
then return 0;
end;
redis.call('expire', KEYS[1], tonumber(ARGV[2]));
return 1;
```

### 误解锁情况

如果线程A获取到了锁并且设置了超时时间为5S，然后执行开始执行业务代码。但是如果到了超时时间业务还没执行完，锁到了超时时间此时锁并解除了，同时另外一个线程B获取到了锁，开始执行代码。而此刻线程A业务代码执行完成后解除了锁。

上面的场景存在两个问题：

- **资源没锁住**：线程A还没有执行完，线程B就开始执行了，很显然资源没有锁住，并发安全得不到保证。

- **释放了不属于自己的锁**：线程A的锁因为超时被释放从而释放了线程B的锁，这显然是不合理的。

针对上面两个问题又该如何解决呢？

#### 锁的超时时间

锁的超时时间小于业务执行时间时，锁因为过期被释放了，那么我们该如何确定锁的超时时间呢？很显然不同的业务这个锁的时间并不确定，我们也无法保证超时时间内业务一定执行完了。如果设置的太大，这将程序未主动释放锁时，其他线程等待锁的时间太久。

对于这种情况，通常我们的解决方案是在线程获取锁后增加守护线程，为将要过期但未释放的锁去增加有效时间。

#### 锁误解除

锁误解除是指线程解除了不是自己加的锁，在上面的情况中可能会发生。要解决这个问题，通常情况我们在加锁时为锁设置一个唯一值，然后再解锁时判断key的值与我们之前加锁时的值是否一致，如果一致则解锁成功。Redis中并未提供该命令，不过我们可以使用Lua脚本来实现。

```lua
if redis.call("get",KEYS[1]) == ARGV[1]
then
    return redis.call("del",KEYS[1])
else
    return 0
end
```

## 可重入锁

通过上面的讲解我们知道了实现Redis锁的核心逻辑，但是上面实现的锁没法实现可重入性。这里简单的讲一些什么是可重入性。

```java
public class App {
    public static void main(String[] args) {
        a();
    }

    public static synchronized void a(){
        System.out.println("线程进入a方法");
        b();
    }

    public static synchronized void b(){
        System.out.println("线程进入b方法");
    }
}
```

上面是可重入性最简单的示例，方法a和b都是用了**synchronized**修饰，因为是静态方法它们锁的是同一资源及**App.class**，主线程进入a方法获取到了锁，a方法中调用b再次获取到了锁，而这两个方法获取都是同一把锁，而这个特性就是锁的可重入性。

在前面我们实现的可重入锁很显然是无法实现可重入性的，因为同一线程再次获取锁会失败。重入锁要解决的核心问题就是，对于同一个线程再次获取锁要能成功同时能记住获取锁的次数，而每次解锁会将加锁次数减一。如果解锁次数为0时，锁被完全解除，需要删除Redis中的锁。总结起来就是以下两点核心要求：

- 同一线程能多次获取锁。

- 需要记住获取锁的次数。

通常情况下实现Redis分布锁的可重入特性有两种方案：

- ThreadLocal实现。

- Redis Hash实现。

### ThreadLocal实现

Java中的ThreadLocal可以使每个线程拥有一个自己的存储空间，而这个存储空间中的数据只有当前线程能访问，所以不存在线程安全问题。这里使用伪代码写一下如何实现。

```java
//用来存储当前线程获取锁的次数
ThreadLocal<Map<String, Integer>> LOCKS = ThreadLocal.withInitial(HashMap::new);


//获取锁
boolean getLock(String lockName){
    //获取当前线程锁的重入次数。其中Map的key存的是锁名称，value存储的是获取锁的次数。
    //为什么使用Map结构是因为线程可能还会获取别的锁。
    Map<String,Integer> counts = LOCKS.get();
    if(counts.containsKey(lockName)){
        //说明之前已经获取过该锁了,将获取锁的次数+1
        counts.put(lockName, counts.get(lockName) + 1);
        return true;
    } else {
        //第一次获取锁
        if(redisLock.lock(lockName)){
            //获取到了锁
            counts.put(lockName,1);
            return true;
        }
        //未获取到锁
        return false;
    }
}


//释放锁
void unLock(String lockName){
    //获取当前线程的锁次数
    Map<String,Integer> counts = LOCKS.get();
    if(counts.get(lockNam) <= 1){
        //说明是最后一次释放锁
        redisLock.unLock(lockName);
        //移出计数器
        LOCKS.remove(lockName);
    } else {
        //说明还未完全解锁
        counts.put(lockName,counts.get(lockName) - 1 );
    }
}
```

上面只是使用伪代码写了核心逻辑，但是如果要考虑到其他问题代码会更加复杂。

### Redis Hash实现

使用ThreadLocal实现可重入性主要就是为了记录重入次数。使用String实现Redis分布式锁我们没法记住重入次数，但是Redis提供了Hash这种数据结果可以存储键值对数据结构，可以通过Hash来保存重入次数。下面我们使用Lua脚本来实现加锁过程：

```lua
-- KYES[1]代表锁名
-- ARGV[1]代表过期时间
-- ARGV[2]代表锁值

-- 锁不存在 第一次加锁
if (redis.call('exists', KEYS[1]) == 0) then
    -- hash的key为锁值，value为加锁次数
    redis.call('hincrby', KEYS[1], ARGV[2], 1);
    -- 设置锁的过期时间
    redis.call('pexpire', KEYS[1], ARGV[1]);
    return 1;
end ;
-- 锁已存在
if (redis.call('hexists', KEYS[1], ARGV[2]) == 1) then
    -- 修改加锁次数
    redis.call('hincrby', KEYS[1], ARGV[2], 1);
    -- 重新设置过期时间
    redis.call('pexpire', KEYS[1], ARGV[1]);
    return 1;
end ;
-- 未获取到锁
return 0;
```

- 如果锁不存在，则创建一个key为锁名的hash结构，然后这个hash结构中有一个k-v结构的数据。k为锁的值，v为加锁次数。然后为这个hash结构设置过期时间。

- 锁名存在切对应的锁值存在，则说明是同一个线程再次重入。此时修改重入次数并且刷新过期时间。

- 未获取到锁

上面是加锁逻辑，下面看结果逻辑：

```lua
-- KEYS[1]代表锁名
-- ARGV[1]代表锁值
if (redis.call('hexists', KEYS[1], ARGV[1]) == 0) then
    return nil;
end ;
-- 计算当前可重入次数
local counter = redis.call('hincrby', KEYS[1], ARGV[1], -1);
-- 小于等于 0 代表可以解锁
if (counter > 0) then
    return 0;
else
    redis.call('del', KEYS[1]);
    return 1;
end ;
return nil;
```

- 先判断锁是否存在，不存在直接返回

- 存在则将计数值减一。

- 如果计数值大于0说明还没有完全解锁，等于0时说明锁已全部解除，删除锁即可。

## RedLock

如果真的使用了Redis来实现分布式锁，那么单节点的Redis可靠性是没有保障的，只要Redis挂了，那么分布式锁都将无法正常使用。通常情况下我们会Redis集群，但是主从复制是异步进行的，只要主节点挂了就可能会存在部分数据丢失。为了保证集群下分布式锁的可靠性，Redis官方推荐使用RedLock来解决。

RedLock的核心思想是让客户端与多个独立的Redis节点依次请求申请加锁，如果客户端能够和半数以上（N/2+1）的节点成功完成加锁操作，那么我们就认为客户端能成功的获得分布式锁，否则加锁失败。

## 总结

本文只是大致介绍了Redis实现分布式锁的一些核心点，但如果真的要自己使用Redis来实现分布式锁还是有一定难度的，可能写的代码问题也比较多。如果你是使用JAVA开发，强烈推荐你使用**redisson**工具，这个工具包提供多种不同的分布式锁，例如可重入锁、公平锁、读写锁、RedLock等。具体可以参照下面链接：

```url
https://github.com/redisson/redisson/wiki/8.-distributed-locks-and-synchronizers
```

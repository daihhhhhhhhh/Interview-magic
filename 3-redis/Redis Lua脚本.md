# [redis分布式锁和lua脚本](https://www.cnblogs.com/number7/p/8320259.html)

**业务背景：存储请求参数token ，token唯一 ，且新的生成旧的失效**

思路：因为是多台机器，获取token存入redis，保持唯一，考虑使用redis来加锁，其实就是在redis中存一个key，其他机器发现key有值的话就不进行获取token的请求。

SET操作会覆盖原有值，SETEX虽然可设置key过期时间，但也会覆盖原有值，所以考虑可以使用SETNX

```
SETNX Key value
```

将 key 的值设为 value ，当且仅当 key 不存在。

 

若给定的 key 已经存在，则 SETNX 不做任何动作

 

成功返回1，失败返回0。

### 看上去SETNX 配合 EXPIRE（过期时间）是个不错的选择，于是就有了加锁错误示例1：

```
jedis.setnx("lockName","value");
//这里redis挂掉，就是一个死锁
jedis.expire("lockName",10);
```

因为这两个操作不具备原子性，所以可能出现死锁，之所以有这样的示例，是因为低版本的redis的SET还不支持多参数命令

```
`从 Redis ``2.6``.``12` `版本开始， SET 命令的行为可以通过一系列参数来修改``EX second ：设置键的过期时间为 second 秒。 SET key value EX second 效果等同于 SETEX key second value 。``PX millisecond ：设置键的过期时间为 millisecond 毫秒。 SET key value PX millisecond 效果等同于 PSETEX key millisecond value 。``NX ：只在键不存在时，才对键进行设置操作。 SET key value NX 效果等同于 SETNX key value 。``XX ：只在键已经存在时，才对键进行设置操作。`
```

这里可以引出 redis正确的加锁示例：

```
`public` `static` `boolean` `lock(Jedis jedis, String lockKey, String uid, ``int` `expireTime) { `` ` `    ``String result = jedis.set(lockKey, uid,``"NX"` `"PX"``, expireTime); `` ` `    ``if` `(``"OK"``.equals(result)) { ``      ``return` `true``; ``    ``} ``    ``return` `false``; `` ` `  ``}`
```

　其实就等于在redis中执行了 ：`set key value nx px 10000`

![img](https://images2018.cnblogs.com/blog/763252/201806/763252-20180616192817868-1574021556.png)

第一个为key，我们使用key来当锁名

第二个为value，我们传的是uid，唯一随机数，也可以使用本机mac地址 + uuid

第三个为NX，意思是SET IF NOT EXIST，即当key不存在时，我们进行set操作；若key已经存在，则不做任何操作 第四个为PX，意思是我们要给这个key加一个过期的设置，具体时间由第五个参数决定 第五个为time，代表key的过期时间，对应第四个参数 PX毫秒，EX秒

 

再来看一下分布式锁的要求：

 

分布式锁是用于解决分布式系统中操作共享资源时的数据一致性问题

 

为了确保分布式锁可用，我们至少要确保锁的实现同时满足以下四个条件：

 

#### 互斥性。在任意时刻，只有一个客户端能持有锁。

#### 不会发生死锁。即使有一个客户端在持有锁的期间崩溃而没有主动解锁，也能保证后续其他客户端能加锁。

#### 性能。排队等待锁的节点如果不知道锁何时会被释放，则只能隔一段时间尝试获取一次锁，这样无法保证资源的高效利用，因此当锁释放时，要能够通知等待队列，使一个等待节点能够立刻获得锁。

#### 重入。同一个线程可以重复拿到同一个资源的锁。

NX保证互斥性

PX保证不会死锁

Value传入的唯一标识保证是自己的锁（可以通过随机uuid+线程名称 来保证唯一）

#### PS:因为 SET 命令可以通过参数来实现和 SETNX 、 SETEX 和 PSETEX 三个命令的效果，不知道将来的 Redis 版本会不会废弃 SETNX 、 SETEX 和 PSETEX 这三个命令 ？

 

**下面看一个释放锁的错误示例**：

```
`public` `static` `void` `wrongUnLock1(Jedis jedis, String lockKey, String requestId) { `` ` `  ``// 判断加锁与解锁是不是同一个线程 ``  ``if` `(requestId.equals(jedis.get(lockKey))) { ``    ``// lockkey锁失效，下一步删除的就是别人的锁 ``    ``jedis.del(lockKey); ``  ``} `` ` `} 　`
```

根本问题还是保证操作的原子性，因为是两步操作，即便判断到是当前线程的锁，但是也有可能再删除之前刚好过期，这样删除的就是其他线程的锁。


**如果业务要求精细，我们可以使用lua脚本来进行完美解锁**


```
/**
     * redis可以保证lua中的键的原子操作 unlock:lock调用完之后需unlock,否则需等待lock自动过期
     *
     * @param lock
     *  uid 只有线程已经获取了该锁才能释放它（uid相同表示已获取）
     */
    public  void unlock( String lock) {

        Jedis jedis = new Jedis("localhost");
       
        final String uid= tokenMap.get();
        if (StringUtil.isBlank(token))
            return;
        try {
            final String script = "if redis.call(\"get\",\"" + lock + "\") == \"" +
             uid + "\"then  return redis.call(\"del\",\"" + lock + "\") else return 0 end ";
            jedis.eval(script);
        } catch (Exception e) {
            throw new RedisException("error");
        } finally {
            if (jedis != null)
                jedis.close();
        }
    }
```




关于lua：

**Lua 是一种轻量小巧的脚本语言，用标准C语言编写并以源代码形式开放， 其设计目的是为了嵌入应用程序中，从而为应用程序提供灵活的扩展和定制功能。**

**Lua 提供了交互式编程模式。我们可以在命令行中输入程序并立即查看效果。**

 



### lua脚本优点

- 减少网络开销：本来多次网络请求的操作，可以用一个请求完成，原先多次请求的逻辑放在redis服务器上完成。使用脚本，减少了网络往返时延
- 原子操作：Redis会将整个脚本作为一个整体执行，中间不会被其他命令插入
- 复用：客户端发送的脚本会永久存储在Redis中，意味着其他客户端可以复用这一脚本而不需要使用代码完成同样的逻辑

 上面这个脚本很简单

```
if redis.call(\"get\",\"" + lock + "\") // redisGET命令
 == \"" +uid + // 判断是否是当前线程
 "\"then  return redis.call(\"del\",\"" + lock + "\") // 如果是，执行redis DEL操作，删除锁
 else return 0 end  
```

 

**同理我们可以使用lua给线程加锁**

 

```
local lockkey = KEYS[1]
--唯一随机数
local uid = KEYS[2]
--失效时间，如果是当前线程，也是续期时间
local time = KEYS[3]

if redis.call('set',lockkey,uid,'nx','px',time)=='OK' then
return 'OK'
else
    if redis.call('get',lockkey) == uid then
       if redis.call('EXPIRE',lockkey,time/1000)==1 then
       return 'OOKK'
       end
    end
end
```




lua脚本也可以通过外部文件读取，方便修改

 


```
  public void luaUnLock() throws Exception{
        Jedis jedis = new Jedis("localhost") ;
        InputStream input = new FileInputStream("unLock.lua");
        byte[] by = new byte[input.available()];
        input.read(by);
        String script = new String(by);
        Object obj = jedis.eval(script, Arrays.asList("key","123"), Arrays.asList(""));
        System.out.println("执行结果 " + obj);
    }
```



 

PS：跟同事讨论的时候，想到可不可以利用redis的额事物来解锁，并没有实际使用，怕有坑。

### redis事物解锁



```
public boolean unLock(Jedis jedis, String lockName, String uid) throws Exception{
            jedis.watch(lockName);
            //这里的判断uid和下面的del虽然不是原子性，有了watch可以保证不会误删锁
            if (jedis.get(lockName).equals(uid)) {
                redis.clients.jedis.Transaction transaction = jedis.multi();
                transaction.del(lockName);
                List<Object> exec = transaction.exec();
                if (exec.get(0).equals("OK")) {
                    transaction.close();
                    return true;
                }
            }
            return false;
        }
```



 

 

### 可重入锁

可重入锁，也叫做递归锁，指的是同一线程 外层函数获得锁之后 ，内层递归函数仍然有获取该锁的代码，但不受影响。

 

在Java中用set命令实现可重入锁

 



```
//保存每个线程独有的token
    private static ThreadLocal<String> tokenMap = new ThreadLocal<>();

/**
     * 这个例子还不太完善。  
     * redis实现分布式可重入锁,并不保证在过期时间内完成锁定内的任务，需根据业务逻辑合理分配seconds
     *
     * @param lock
     *            锁的名称
     * @param mseconds
     *            锁定时间，单位 毫秒
     *  token 对于同一个lock,相同的token可以再次获取该锁，不相同的token线程需等待到unlock之后才能获取
     *
     */
    public  boolean lock(final String lock,  int mseconds ,Jedis jedis) {
        // token 对于同一个lock,相同的token可以再次获取该锁，不相同的token线程需等待到unlock之后才能获取
        String token = tokenMap.get();
        if (StringUtil.isBlank(token)) {
            token = UUID.randomUUID().toString().replaceAll("-","");
            tokenMap.set(token);
        }
        boolean flag = false;
        try {
            String ret = jedis.set(lock, token, "NX", "PX", mseconds);
            if (ret == null) {// 该lock的锁已经存在
                String origToken = jedis.get(lock);// 即使lock已经过期也可以
                if (token.equals(origToken) || origToken==null) {
                // token相同默认为同一线程，所以token应该尽量长且随机，保证不同线程的该值不相同
                    ret = jedis.set(lock, token, "NX", "PX", mseconds);//
                    if ("OK".equalsIgnoreCase(ret))
                        flag = true;
                    System.out.println("当前线程 " + token);
                }
            } else if ("OK".equalsIgnoreCase(ret))
                flag = true;
            System.out.println("当前线程 " + token);
        } catch (Exception e) {

        } finally {
            if (jedis != null)
                jedis.close();
        }
        return flag;
    }
```



 

继续正题，说到lua脚本 和 可重入锁，就不得不提 redission了

## redission

redisson是redis官网推荐的java语言实现分布式锁的项目

redission中提供了多样化的锁，

### 可重入锁（Reentrant Lock）

### 公平锁（Fair Lock）

### 联锁（MultiLock）

### 红锁（RedLock）

###  读写锁（ReadWriteLock）

### 信号量（Semaphore） 等等

下面分析一下可重入的源码



```
 /**
     * redission分布式锁-重试时间 秒为单位
     * @param lockName 锁名
     * @param waitTime  重试时间
     * @param leaseTime 锁过期时间
     * @return
     */
    public boolean tryLock(String lockName,long waitTime,long leaseTime){
        try{
            RLock rLock = redissonClient.getLock(lockName);
            return rLock.tryLock(waitTime, leaseTime, TimeUnit.SECONDS);
        }catch (Exception e){
            logger.error("redission lock error with waitTime",e);
        }
        return false;
    }
```



#### org.redisson.Redisson#getLock()

```
`@Override``public` `RLock getLock(String name) {`` ``return` `new` `RedissonLock(commandExecutor, name, id);``}`
```

　　

- commandExecutor: 与 Redis 节点通信并发送指令的真正实现。需要说明一下，Redisson 的 CommandExecutor 实现是通过 eval 命令来执行 Lua 脚本，所以要求 Redis 的版本必须为 2.6 或以上
- name: 锁的全局名称，例如上面代码中的 "foobar"，具体业务中通常可能使用共享资源的唯一标识作为该名称。
- id: Redisson 客户端唯一标识。

 

#### org.redisson.RedissonLock#lock()

在直接使用 lock() 方法获取锁时，最后实际执行的是 lockInterruptibly(-1, null)



```
@Override
public void lockInterruptibly(long leaseTime, TimeUnit unit) throws InterruptedException {
    // 1.尝试获取锁
    Long ttl = tryAcquire(leaseTime, unit);
    // 2.获得锁成功
    if (ttl == null) {
        return;
    }
    // 3.等待锁释放，并订阅锁
    long threadId = Thread.currentThread().getId();
    Future<RedissonLockEntry> future = subscribe(threadId);
    get(future);

    try {
        while (true) {
            // 4.重试获取锁
            ttl = tryAcquire(leaseTime, unit);
            // 5.成功获得锁
            if (ttl == null) {
                break;
            }
            // 6.等待锁释放
            if (ttl >= 0) {
                getEntry(threadId).getLatch().tryAcquire(ttl, TimeUnit.MILLISECONDS);
            } else {
                getEntry(threadId).getLatch().acquire();
            }
        }
    } finally {
        // 7.取消订阅
        unsubscribe(future, threadId);
    }
}
```



 

 

1. 首先尝试获取锁，具体代码下面再看，返回结果是已存在的锁的剩余存活时间，为 null 则说明没有已存在的锁并成功获得锁。
2. 如果获得锁则结束流程，回去执行业务逻辑。
3. 如果没有获得锁，则需等待锁被释放，并通过 Redis 的 channel 订阅锁释放的消息
4. 订阅锁的释放消息成功后，进入一个不断重试获取锁的循环，循环中每次都先试着获取锁，并得到已存在的锁的剩余存活时间。
5. 如果在重试中拿到了锁，则结束循环，跳过第 6 步。
6. 如果锁当前是被占用的，那么等待释放锁的消息，具体实现使用了 JDK 并发的信号量工具 Semaphore 来阻塞线程，当锁释放并发布释放锁的消息后，信号量的 release() 方法会被调用，此时被信号量阻塞的等待队列中的一个线程就可以继续尝试获取锁了。
7. 在成功获得锁后，就没必要继续订阅锁的释放消息了，因此要取消对 Redis 上相应 channel 的订阅。

 

#### 重点看一下 tryAcquire() 方法的实现



```
private Long tryAcquire(long leaseTime, TimeUnit unit) {
    return get(tryAcquireAsync(leaseTime, unit, Thread.currentThread().getId()));
}

private <T> Future<Long> tryAcquireAsync(long leaseTime, TimeUnit unit, long threadId) {
    if (leaseTime != -1) {
        return tryLockInnerAsync(leaseTime, unit, threadId, RedisCommands.EVAL_LONG);
    }
    // 2.用默认的锁超时时间去获取锁
    Future<Long> ttlRemainingFuture = tryLockInnerAsync(LOCK_EXPIRATION_INTERVAL_SECONDS,
                TimeUnit.SECONDS, threadId, RedisCommands.EVAL_LONG);
    ttlRemainingFuture.addListener(new FutureListener<Long>() {
        @Override
        public void operationComplete(Future<Long> future) throws Exception {
            if (!future.isSuccess()) {
                return;
            }
            Long ttlRemaining = future.getNow();
            // 成功获得锁
            if (ttlRemaining == null) {
                // 3.锁过期时间刷新任务调度
                scheduleExpirationRenewal();
            }
        }
    });
    return ttlRemainingFuture;
}

<T> Future<T> tryLockInnerAsync(long leaseTime, TimeUnit unit, long threadId,
                RedisStrictCommand<T> command) {
    internalLockLeaseTime = unit.toMillis(leaseTime);
    // 3.使用 EVAL 命令执行 Lua 脚本获取锁
    return commandExecutor.evalWriteAsync(getName(), LongCodec.INSTANCE, command,
              "if (redis.call('exists', KEYS[1]) == 0) then " +
                  "redis.call('hset', KEYS[1], ARGV[2], 1); " +
                  "redis.call('pexpire', KEYS[1], ARGV[1]); " +
                  "return nil; " +
              "end; " +
              "if (redis.call('hexists', KEYS[1], ARGV[2]) == 1) then " +
                  "redis.call('hincrby', KEYS[1], ARGV[2], 1); " +
                  "redis.call('pexpire', KEYS[1], ARGV[1]); " +
                  "return nil; " +
              "end; " +
              "return redis.call('pttl', KEYS[1]);",
                Collections.<Object>singletonList(getName()), internalLockLeaseTime,
                        getLockName(threadId));
}
```



- 获取锁真正执行的命令，Redisson 使用 EVAL 命令执行上面的 Lua 脚本来完成获取锁的操作
- 通过 exists 命令发现当前 key 不存在，即锁没被占用，则执行 hset 写入 Hash 类型数据 key:全局锁名称（例如共享资源ID）, field:锁实例名称（Redisson客户端ID:线程ID）, value:1，并执行 pexpire 对该 key 设置失效时间，返回空值 nil，至此获取锁成功
- 如果通过 hexists 命令发现 Redis 中已经存在当前 key 和 field 的 Hash 数据，说明当前线程之前已经获取到锁，因为这里的锁是可重入的，则执行 hincrby 对当前 key field 的值加一，并重新设置失效时间，返回空值，至此重入获取锁成功。
- 最后是锁已被占用的情况，即当前 key 已经存在，但是 Hash 中的 Field 与当前值不同，则执行 pttl 获取锁的剩余存活时间并返回，至此获取锁失败。

 redisson释放锁



```
public void unlock() {
    // 1.通过 EVAL 和 Lua 脚本执行 Redis 命令释放锁
    Boolean opStatus = commandExecutor.evalWrite(getName(), LongCodec.INSTANCE,
                    RedisCommands.EVAL_BOOLEAN,
                    "if (redis.call('exists', KEYS[1]) == 0) then " +
                        "redis.call('publish', KEYS[2], ARGV[1]); " +
                        "return 1; " +
                    "end;" +
                    "if (redis.call('hexists', KEYS[1], ARGV[3]) == 0) then " +
                        "return nil;" +
                    "end; " +
                    "local counter = redis.call('hincrby', KEYS[1], ARGV[3], -1); " +
                    "if (counter > 0) then " +
                        "redis.call('pexpire', KEYS[1], ARGV[2]); " +
                        "return 0; " +
                    "else " +
                        "redis.call('del', KEYS[1]); " +
                        "redis.call('publish', KEYS[2], ARGV[1]); " +
                        "return 1; "+
                    "end; " +
                    "return nil;",
                    Arrays.<Object>asList(getName(), getChannelName()), 
                            LockPubSub.unlockMessage, internalLockLeaseTime, 
                            getLockName(Thread.currentThread().getId()));
    // 2.非锁的持有者释放锁时抛出异常
    if (opStatus == null) {
        throw new IllegalMonitorStateException(
                "attempt to unlock lock, not locked by current thread by node id: "
                + id + " thread-id: " + Thread.currentThread().getId());
    }
    // 3.释放锁后取消刷新锁失效时间的调度任务
    if (opStatus) {
        cancelExpirationRenewal();
    }
```



1. 使用 EVAL 命令执行 Lua 脚本来释放锁：
2. key 不存在，说明锁已释放，直接执行 publish 命令发布释放锁消息并返回 1。
3. key 存在，但是 field 在 Hash 中不存在，说明自己不是锁持有者，无权释放锁，返回 nil。
4. 因为锁可重入，所以释放锁时不能把所有已获取的锁全都释放掉，一次只能释放一把锁，因此执行 hincrby 对锁的值减一。
5. 释放一把锁后，如果还有剩余的锁，则刷新锁的失效时间并返回 0；如果刚才释放的已经是最后一把锁，则执行 del 命令删除锁的 key，并发布锁释放消息，返回 1。
6. 上面执行结果返回 nil 的情况（即第2中情况），因为自己不是锁的持有者，不允许释放别人的锁，故抛出异常。
7. 执行结果返回 1 的情况，该锁的所有实例都已全部释放，所以不需要再刷新锁的失效时间。

可以看到redission最终还是使用了lua脚本来加解锁 ：

加锁脚本



```
 if (redis.call('exists' KEYS[1]) == 0) then  +  --  exists 判断key是否存在
                  redis.call('hset' KEYS[1] ARGV[2] 1);  +   --如果不存在，hset存哈希表
                  redis.call('pexpire' KEYS[1] ARGV[1]);  + --设置过期时间
                  return nil;  +                            -- 返回null 就是加锁成功
              end;  +
              if (redis.call('hexists' KEYS[1] ARGV[2]) == 1) then  + -- 如果key存在，查看哈希表中是否存在
                  redis.call('hincrby' KEYS[1] ARGV[2] 1);  + -- 给哈希中的key加1，代表重入1次，以此类推
                  redis.call('pexpire' KEYS[1] ARGV[1]);  + -- 重设过期时间
                  return nil;  +
              end;  +
              return redis.call('pttl' KEYS[1]); --如果前面的if都没进去，说明ARGV2 的值不同，也就是不是同                  一线程的锁，这时候直接返回该锁的过期时间
```



推荐使用sciTE来编辑lua

![img](https://images2018.cnblogs.com/blog/763252/201806/763252-20180616204241244-1695484151.png)

解锁的脚本就不分析了，还是操作的redis命令，主要是lua脚本执行的时候能保证原子性。

### lua脚本的缺点

Redis的脚本执行是原子的，即脚本执行期间Redis不会执行其他命令。所有的命令都必须等待脚本执行完成后才能执行。为了防止某个脚本执行时间过长导致Redis无法提供服务（比如陷入死循环），Redis提供了lua-time-limit参数限制脚本的最长运行时间，默认为5秒钟。当脚本运行时间超过这一限制后，Redis将开始接受其他命令但不会执行（以确保脚本的原子性，因为此时脚本并没有被终止），而是会返回“BUSY”错误

 

一个lua死循环脚本

```
`a = ``0``while``(a < ``3``) ``do``print(``"x = "` `.. ``'我是循环'``)``end`
```

　　

 

几个lua脚本示例

#### 示例1——实现访问频率限制: 实现访问者 $ip 在一定的时间 time 内只能访问 limit 次



```
local key = "rate.limit:" .. KEYS[1]
local limit = tonumber(ARGV[1])
local expire_time = ARGV[2]



local is_exists = redis.call("EXISTS", key)
if is_exists == 1 then
    if redis.call("INCR", key) > limit then
        return '拒绝访问'
    else
        return '可以访问'
    end
else
    return redis.call("SET", key, "1","NX","PX",expire_time)
end
```



### 示例2 —— 抢红包

 



```
-- 脚本：尝试获得红包，如果成功，则返回json字符串，如果不成功，则返回空
 -- 参数：红包队列名， 已消费的队列名，去重的Map名，用户ID
 -- 返回值：nil 或者 json字符串，包含用户ID：userId，红包ID：id，红包金额：money
 --  jedis.eval(getScript(), 4, hongBaoList, hongBaoConsumedList, hongBaoConsumedMap, "" + j)
        if redis.call('hexists', KEYS[3], KEYS[4]) ~= 0 then
              return nil
              else
              -- 先取出一个小红包
              local hongBao = redis.call('rpop', KEYS[1])
              -- hongbao ： {"Money":9,"Id":8}
                  if hongBao then
                      local x = cjson.decode(hongBao)
                      -- 加入用户ID信息
                      x['userId'] = KEYS[4]
                      local re = cjson.encode(x)
                      -- 把用户ID放到去重的set里
                      redis.call('hset', KEYS[3], KEYS[4], KEYS[4])
                      -- 把红包放到已消费队列里
                      redis.call('lpush', KEYS[2], re)
                  return re;
                  end
              end
          return nil
```

红包队列：hongBaoList -- keys1

已消费hash表 ：hongBaoConsumedMap -- key2

已消费队列：hongBaoConsumedList -- key3

#### SCRIPT LOAD 命令

```
`redis ``127.0``.``0.1``:``6379``> SCRIPT LOAD ``"return 'hello moto'"``"232fd51614574cf0867b83d384a5e898cfd24e5a"` `redis ``127.0``.``0.1``:``6379``> EVALSHA ``"232fd51614574cf0867b83d384a5e898cfd24e5a"` `0``"hello moto"`
```

　　

目的：在脚本比较长的情况下，如果每次调用脚本都需要将整个脚本传给Redis会占用较多的带宽。为了解决这个问题，Redis提 供 了EVALSHA命令，允许开发者通过脚本内容的SHA1摘要来执行脚本

 

 

- SCRIPT LOAD "lua-script"   获得 脚本的SHA1摘要 lua-script-sha1

- SCRIPT EXISTS lua-script-sha1  可以判断脚本是否存在 存在-1， 不存在-0

- SCRIPT FLUSH 清空 SHA1摘要  redis将脚本的SHA1摘要加入到脚本缓存后会永久保留，不会删除，但可以手动使用SCRIPT FLUSH命令情况脚本缓存

- SCRIPT KILL 强制终止当前脚本

#### Java中使用 SCRIPT LOAD

```
public void scriptLoad()throws Exception{
        Jedis jedis = new Jedis("localhost");
        //从文件读取lua脚本
        InputStream input = new FileInputStream("return.lua");
        byte[] by = new byte[input.available()];
        input.read(by);
        byte[] scriptBy = jedis.scriptLoad(by);
        String sha1 = new String(scriptBy);
        //直接解析
        String sha2 = jedis.scriptLoad("local key1 = KEYS[1]\n"
            + "local key2 = KEYS[2]\n"
            + "local argv1 = ARGV[1]\n"
            + "return \"key1:\"..key1 ..\" key2:\"..key2.. \" argv1:\"..argv1");
        System.out.println("sha1 : " + sha1);
        System.out.println("sha2 : " + sha2);
        Object obj = jedis.evalsha(sha1, Arrays.asList("value1","value2"), Arrays.asList("value3"));
        System.out.println("执行结果： "+ obj);
    }
```

 

\~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

主要内容是以上这些，写的不好，如果大家发现bug，请@我 

20:49:54

最后是自己学习lua的一些笔记，含金量不高

 

## lua简介

Lua 是一种轻量小巧的脚本语言，用标准C语言编写并以源代码形式开放， 其设计目的是为了嵌入应用程序中，从而为应用程序提供灵活的扩展和定制功能。

Lua 提供了交互式编程模式。我们可以在命令行中输入程序并立即查看效果。

 

> print("Hello lua！")

 

Hello lua！

 

单行注释 -- 多行注释 --[[ 注释 注释 --]]

 

#### 标示符

Lua 标示符用于定义一个变量，函数获取其他用户定义的项。标示符以一个字母 A 到 Z 或 a 到 z 或下划线 _ 开头后加上0个或多个字母，下划线，数字

Lua 不允许使用特殊字符如 @, $, 和 % 来定义标示符。 Lua 是一个区分大小写的编程语言。

#### 关键词

以下列出了 Lua 的保留关键字。保留关键字不能作为常量或变量或其他用户自定义标示符：

 

and break do else elseif end false for function if in local nil not or repeat return then true until while 一般约定，以下划线开头连接一串大写字母的名字（比如 _VERSION）被保留用于 Lua 内部全局变量。

 

 

#### lua数据类型

 

Lua是动态类型语言，变量不要类型定义,只需要为变量赋值，8个基本类型

nil 这个最简单，只有值nil属于该类，表示一个无效值 boolean 包含两个值：false和true。 number 表示双精度类型的实浮点数 string 字符串由一对双引号或单引号来表示 function 由 C 或 Lua 编写的函数 userdata 表示任意存储在变量中的C数据结构 thread 表示执行的独立线路，用于执行协同程序 table Lua 中的表（table）其实是一个"关联数组"（associative arrays），数组的索引可以是数字或者是字符串。在 Lua 里，table 的创建是通过"构造表达式"来完成，最简单构造表达式是{}，用来创建一个空表。

 

可以使用tpye() 获得类型。

 

变量不需要声明

```
print(b)
nil

b=10
print(b)
10
```

 

Lua 中的变量全是全局变量，那怕是语句块或是函数里，除非用 local 显式声明为局部变量。

a = 5        -- 全局变量

local b = 5     -- 局部变量

 

#### lua数据类型的自动转换

运行时，Lua会自动在string和numbers之间自动进行类型转换，当一个字符串使用算术操作符时， string 就会被转成数字。

print("10"+ 1)   --> 11

print("a"+ 1)  --> error

 

```
##function
	function fun1(n)
    if n == 0 then
        return 1
    else
        return n * fun1(n - 1)
    end
	end
	print(fun1(3))
	fun2 = fun1
	print(fun2(4))
```

 

 

#### table 与Java不同，Lua是从1开始排序

 

```
tab = {"Hello","World","hello","lua"}
for k,v in pairs(tab) do
print(k.." "..v)
end
```

 

```
##for循环
for i=1,10
do print(i)
end
```

 

#### if判断

Lua中 0 为 true ， nil

 

```
--[ 0 为 true ]
if(0)
then
print("0 为 true")
else
print("0 为 false")
end


if(null)
then
print("nil 为 true")
else
print("nil 为 false")
end
```
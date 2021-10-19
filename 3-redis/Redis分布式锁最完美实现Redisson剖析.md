# Redis分布式锁最完美实现Redisson剖析

同一服务在单实例部署环境可通过synchronized、ReentrantLock本地锁来保证在一次只能允许一个线程执行被锁住的代码块，但在分布式多实例部署环境需实现分布式锁来保证互斥性。redission堪称redis分布式锁最完美实现，接下来一起来剖析redisson实现机制。

![img](https://pics1.baidu.com/feed/b999a9014c086e062ec07bafc424cef10bd1cb98.jpeg?token=2ebd4399b1fd13483f4e190357f66036&s=85B3C530DA8D32C8036D94C60300A0A1)

**实现分布式锁需要关注哪些细节呢？**

确保互斥：在同一时刻，必须保证锁至多只能被一个客户端持有。不能死锁：在一个客户端在持有锁的期间崩溃而没有主动解锁情况下，也能保证后续其他客户端能加锁。避免活锁：在获取锁失败的情况下，反复进行重试操作，占用Cpu资源，影响性能。实现更多锁特性：锁中断、锁重入、锁超时等。确保客户端只能解锁自己持有的锁。

**源码地址**

https://github.com/redisson/redisson

**加锁：**

![img](https://pics4.baidu.com/feed/9922720e0cf3d7ca6d9aa4ff3a330b0c6a63a9f9.jpeg?token=4d9b10c900d94586185904242110899f&s=38C2B3409FA4B9704ED14C060000F0C2)

org.redisson.RedissonLock类tryLockInnerAsync通过eval命令执行Lua代码完成加锁操作。KEYS[1]为锁在redis中的key，key对应value为map结构，ARGV[1]为锁超时时间，ARGV[2]为锁value中的key。ARGV[2]由UUID+threadId组成，用来标记锁被谁持有。

(1) 第一个If判断key是否存在，不存在完成加锁操作

redis.call('hset', KEYS[1], ARGV[2], 1);创建key[1] map中添加key：ARGV[2] ，value：1redis.call('pexpire', KEYS[1], ARGV[1]);设置key[1]过期时间，避免发生死锁。eval命令执行Lua代码的时候，Lua代码将被当成一个命令去执行，并且直到eval命令执行完成，Redis才会执行其他命令。可避免第一条命令执行成功第二条命令执行失败导致死锁。

(2)第二个if判断key存在且当前线程已经持有锁, 重入:

redis.call('hexists', KEYS[1], ARGV[2])；判断redis中锁的标记值是否与当前请求的标记值相同，相同代表该线程已经获取锁。redis.call('hincrby', KEYS[1], ARGV[2], 1);记录同一线程持有锁之后累计加锁次数实现锁重入。redis.call('pexpire', KEYS[1], ARGV[1]); 重制锁超时时间。(3)key存在被其他线程获取的锁, 等待:

redis.call('pttl', KEYS[1]);加锁失败返回锁过期时间。**锁等待**

![img](https://pics2.baidu.com/feed/b21bb051f8198618c62044bb83c19b768bd4e659.jpeg?token=e47fc0e479a246cf90bb689a893fd1ca&s=4BC0834613FC936C4458B40E000070C1)

（1）步骤一：调用加锁操作；

（2）步骤二：步骤一中加锁操作失败，订阅消息，利用redis的pubsub提供一个通知机制来减少不断的重试，避免发生活锁。

（3）步骤三：

![img](https://pics5.baidu.com/feed/dbb44aed2e738bd482e3e42069a732d3267ff997.jpeg?token=cce0000cec4af940c6c2ed00ed2b268f&s=F9E29144DFE0B9705275CD0F0000B0C3)

getLath()获取RedissionLockEntry实例latch变量，由于permits为0，所以嗲用acquire()方法后线程阻塞。

**解锁**

![img](https://pics4.baidu.com/feed/4034970a304e251f45a5ba5b60aa7c127e3e53f1.jpeg?token=bbd24691e270887a24876c973c77dea5&s=33C2B3449FA191781EC1E4060000F0C3)

（1）第一个if判断锁对应key是否存在，不存在publish消息，将获取锁被阻塞的线程恢复重新获取锁；

（2）第二个if判断锁对应key存在，value中是否存在当前要释放锁的标示，不存在返回nil，确保锁只能被持有的线程释放。

（3）对应key存在，value中存在当前要释放锁的标示，将锁标示对应值-1，第三个if判断锁标示对应的值是否大于0，大于0，表示有锁重入情况发生，重新设置锁过期时间。

（4）对应key存在，value中存在当前要释放锁的标示，将锁标示对应值-1后等于0，调用del操作释放锁，并publish消息，将获取锁被阻塞的线程恢复重新获取锁；

![img](https://pics4.baidu.com/feed/b90e7bec54e736d1e8d3847e537cfac7d46269b2.jpeg?token=ac9a4818a39686458ba62aa31b81b565&s=F9C0B3461FA0B9685C7D8C0B0000F0C3)

订阅者接收到publish消息后，执行release操作，调用acquire被阻塞的线程将继续执行获取锁操作。
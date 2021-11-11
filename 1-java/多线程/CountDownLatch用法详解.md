# CountDownLatch用法详解

## **1、CountDownLatch 概念**

CountDownLatch可以使一个获多个线程等待其他线程各自执行完毕后再执行。



CountDownLatch 定义了一个计数器，和一个阻塞队列， 当计数器的值递减为0之前，阻塞队列里面的线程处于挂起状态，当计数器递减到0时会唤醒阻塞队列所有线程，这里的计数器是一个标志，可以表示一个任务一个线程，也可以表示一个倒计时器，CountDownLatch可以解决那些一个或者多个线程在执行之前必须依赖于某些必要的前提业务先执行的场景。



## 2、**CountDownLatch实现原理**

**1、创建计数器**

当我们调用CountDownLatch countDownLatch=new CountDownLatch(4) 时候，此时会创建一个AQS的同步队列，并把创建CountDownLatch 传进来的计数器赋值给AQS队列的 state，所以state的值也代表CountDownLatch所剩余的计数次数；

```java
  public CountDownLatch(int count) {
        if (count < 0) throw new IllegalArgumentException("count < 0");
        this.sync = new Sync(count);//创建同步队列，并设置初始计数器值
    }
```



**2、阻塞线程**

当我们调用countDownLatch.wait()的时候，会创建一个节点，加入到AQS阻塞队列，并同时把当前线程挂起。

```java
  public void await() throws InterruptedException {
        sync.acquireSharedInterruptibly(1);
    }
```



判断计数器是技术完毕，未完毕则把当前线程加入阻塞队列

```java
  public final void acquireSharedInterruptibly(int arg)
            throws InterruptedException {
        if (Thread.interrupted())
            throw new InterruptedException();
        //锁重入次数大于0 则新建节点加入阻塞队列，挂起当前线程
        if (tryAcquireShared(arg) < 0)
            doAcquireSharedInterruptibly(arg);
    }
```



构建阻塞队列的双向链表，挂起当前线程

```java
 private void doAcquireSharedInterruptibly(int arg)
        throws InterruptedException {
        //新建节点加入阻塞队列
        final Node node = addWaiter(Node.SHARED);
        boolean failed = true;
        try {
            for (;;) {
                //获得当前节点pre节点
                final Node p = node.predecessor();
                if (p == head) {
                    int r = tryAcquireShared(arg);//返回锁的state
                    if (r >= 0) {
                        setHeadAndPropagate(node, r);
                        p.next = null; // help GC
                        failed = false;
                        return;
                    }
                }
                //重组双向链表，清空无效节点，挂起当前线程
                if (shouldParkAfterFailedAcquire(p, node) &&
                    parkAndCheckInterrupt())
                    throw new InterruptedException();
            }
        } finally {
            if (failed)
                cancelAcquire(node);
        }
    }
```



**3、计数器递减**

当我们调用countDownLatch.down()方法的时候，会对计数器进行减1操作，AQS内部是通过释放锁的方式，对state进行减1操作，当state=0的时候证明计数器已经递减完毕，此时会将AQS阻塞队列里的节点线程全部唤醒。



```java
 public void countDown() {
        //递减锁重入次数，当state=0时唤醒所有阻塞线程
        sync.releaseShared(1);
    }
```



```java
public final boolean releaseShared(int arg) {
        //递减锁的重入次数
        if (tryReleaseShared(arg)) {
            doReleaseShared();//唤醒队列所有阻塞的节点
            return true;
        }
        return false;
    }
 private void doReleaseShared() {
        //唤醒所有阻塞队列里面的线程
        for (;;) {
            Node h = head;
            if (h != null && h != tail) {
                int ws = h.waitStatus;
                if (ws == Node.SIGNAL) {//节点是否在等待唤醒状态
                    if (!compareAndSetWaitStatus(h, Node.SIGNAL, 0))//修改状态为初始
                        continue;
                    unparkSuccessor(h);//成功则唤醒线程
                }
                else if (ws == 0 &&
                         !compareAndSetWaitStatus(h, 0, Node.PROPAGATE))
                    continue;                // loop on failed CAS
            }
            if (h == head)                   // loop if head changed
                break;
        }
    }
```



## **3、CountDownLatch 常用方法说明**



```text
CountDownLatch(int count); //构造方法，创建一个值为count 的计数器。

await();//阻塞当前线程，将当前线程加入阻塞队列。

await(long timeout, TimeUnit unit);//在timeout的时间之内阻塞当前线程,时间一过则当前线程可以执行，

countDown();//对计数器进行递减1操作，当计数器递减至0时，当前线程会去唤醒阻塞队列里的所有线程。
```

## **4、用CountDownLatch 来优化我们的报表统计**



**功能现状**

运营系统有统计报表、业务为统计每日的用户新增数量、订单数量、商品的总销量、总销售额......等多项指标统一展示出来，因为数据量比较大，统计指标涉及到的业务范围也比较多，所以这个统计报表的页面一直加载很慢，所以需要对统计报表这块性能需进行优化。



**问题分析**

统计报表页面涉及到的统计指标数据比较多，每个指标需要单独的去查询统计数据库数据，单个指标只要几秒钟，但是页面的指标有10多个，所以整体下来页面渲染需要将近一分钟。



**解决方案**

任务时间长是因为统计指标多，而且指标是串行的方式去进行统计的，我们只需要考虑把这些指标从串行化的执行方式改成并行的执行方式，那么整个页面的时间的渲染时间就会大大的缩短， 如何让多个线程同步的执行任务，我们这里考虑使用多线程，每个查询任务单独创建一个线程去执行，这样每个统计指标就可以并行的处理了。



**要求**

因为主线程需要每个线程的统计结果进行聚合，然后返回给前端渲染，所以这里需要提供一种机制让主线程等所有的子线程都执行完之后再对每个线程统计的指标进行聚合。 这里我们使用CountDownLatch 来完成此功能。



**模拟代码**

1、分别统计4个指标用户新增数量、订单数量、商品的总销量、总销售额；

2、假设每个指标执行时间为3秒。如果是串行化的统计方式那么总执行时间会为12秒。

3、我们这里使用多线程并行，开启4个子线程分别进行统计

4、主线程等待4个子线程都执行完毕之后，返回结果给前端。

```java

    //用于聚合所有的统计指标
    private static Map map=new HashMap();
    //创建计数器，这里需要统计4个指标
    private static CountDownLatch countDownLatch=new CountDownLatch(4);

    public static void main(String[] args) {

        //记录开始时间
        long startTime=System.currentTimeMillis();

        Thread countUserThread=new Thread(new Runnable() {
            public void run() {
                try {
                    System.out.println("正在统计新增用户数量");
                    Thread.sleep(3000);//任务执行需要3秒
                    map.put("userNumber",1);//保存结果值
                    countDownLatch.countDown();//标记已经完成一个任务
                    System.out.println("统计新增用户数量完毕");
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }

            }
        });
        Thread countOrderThread=new Thread(new Runnable() {
            public void run() {
                try {
                    System.out.println("正在统计订单数量");
                    Thread.sleep(3000);//任务执行需要3秒
                    map.put("countOrder",2);//保存结果值
                    countDownLatch.countDown();//标记已经完成一个任务
                    System.out.println("统计订单数量完毕");
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }

            }
        });

        Thread countGoodsThread=new Thread(new Runnable() {
            public void run() {
                try {
                    System.out.println("正在商品销量");
                    Thread.sleep(3000);//任务执行需要3秒
                    map.put("countGoods",3);//保存结果值
                    countDownLatch.countDown();//标记已经完成一个任务
                    System.out.println("统计商品销量完毕");
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }

            }
        });

        Thread countmoneyThread=new Thread(new Runnable() {
            public void run() {
                try {
                    System.out.println("正在总销售额");
                    Thread.sleep(3000);//任务执行需要3秒
                    map.put("countmoney",4);//保存结果值
                    countDownLatch.countDown();//标记已经完成一个任务
                    System.out.println("统计销售额完毕");
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }

            }
        });
        //启动子线程执行任务
        countUserThread.start();
        countGoodsThread.start();
        countOrderThread.start();
        countmoneyThread.start();

        try {
            //主线程等待所有统计指标执行完毕
            countDownLatch.await();
            long endTime=System.currentTimeMillis();//记录结束时间
            System.out.println("------统计指标全部完成--------");
            System.out.println("统计结果为："+map.toString());
            System.out.println("任务总执行时间为"+(endTime-startTime)/1000+"秒");

        } catch (InterruptedException e) {
            e.printStackTrace();
        }


    }

```

**执行结果**

![img](https://pic3.zhimg.com/80/v2-05fcd26ad1559a71370a32ad4c43887e_720w.jpg)

## 5、CountDownLatch的用法

CountDownLatch典型用法1：某一线程在开始运行前等待n个线程执行完毕。将CountDownLatch的计数器初始化为n new CountDownLatch(n) ，每当一个任务线程执行完毕，就将计数器减1 countdownlatch.countDown()，当计数器的值变为0时，在CountDownLatch上 await() 的线程就会被唤醒。一个典型应用场景就是启动一个服务时，主线程需要等待多个组件加载完毕，之后再继续执行。

CountDownLatch典型用法2：实现多个线程开始执行任务的最大并行性。注意是并行性，不是并发，强调的是多个线程在某一时刻同时开始执行。类似于赛跑，将多个线程放到起点，等待发令枪响，然后同时开跑。做法是初始化一个共享的CountDownLatch(1)，将其计数器初始化为1，多个线程在开始执行任务前首先 coundownlatch.await()，当主线程调用 countDown() 时，计数器变为0，多个线程同时被唤醒。

CountDownLatch原理
CountDownLatch是通过一个计数器来实现的，计数器的初始化值为线程的数量。每当一个线程完成了自己的任务后，计数器的值就相应得减1。当计数器到达0时，表示所有的线程都已完成任务，然后在闭锁上等待的线程就可以恢复执行任务。 



## 6、CountDownLatch使用例子

1

```
public static void main(String[] args) throws InterruptedException {
        CountDownLatch latch = new CountDownLatch(10);

        for (int i=0; i<9; i++) {
            new Thread(new Runnable() {
                @Override
                public void run() {
                    System.out.println(Thread.currentThread().getName() + " 运行");
                    try {
                        Thread.sleep(3000);
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    } finally {
                        latch.countDown();
                    }
                }
            }).start();
        }
     
        System.out.println("等待子线程运行结束");
        latch.await(10, TimeUnit.SECONDS);
        System.out.println("子线程运行结束");
}
```
2、子线程等待主线程处理完毕开始处理，子线程处理完毕后，主线程输出
```

class MyRunnable implements Runnable {

    private CountDownLatch countDownLatch;
     
    private CountDownLatch await;
     
    public MyRunnable(CountDownLatch countDownLatch, CountDownLatch await) {
        this.countDownLatch = countDownLatch;
        this.await = await;
    }
     
    @Override
    public void run() {
        try {
            countDownLatch.await();
            System.out.println("子线程" +Thread.currentThread().getName()+ "处理自己事情");
            Thread.sleep(1000);
            await.countDown();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
     
    }
}
public static void main(String[] args) throws InterruptedException {
        CountDownLatch countDownLatch = new CountDownLatch(1);
        CountDownLatch await = new CountDownLatch(5);

        for (int i=0; i< 5; i++) {
            new Thread(new MyRunnable(countDownLatch, await)).start();
        }
     
        System.out.println("主线程处理自己事情");
        Thread.sleep(3000);
        countDownLatch.countDown();
        System.out.println("主线程处理结束");
        await.await();
        System.out.println("子线程处理完毕啦");
    }
```
在实时系统中的使用场景
实现最大的并行性：有时我们想同时启动多个线程，实现最大程度的并行性。例如，我们想测试一个单例类。如果我们创建一个初始计数器为1的CountDownLatch，并让其他所有线程都在这个锁上等待，只需要调用一次countDown()方法就可以让其他所有等待的线程同时恢复执行。
开始执行前等待N个线程完成各自任务：例如应用程序启动类要确保在处理用户请求前，所有N个外部系统都已经启动和运行了。
死锁检测：一个非常方便的使用场景是你用N个线程去访问共享资源，在每个测试阶段线程数量不同，并尝试产生死锁。

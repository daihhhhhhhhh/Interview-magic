# HashTable

## HashTable操作

HashTable的操作几乎和HashMap一致，主要的区别在于HashTable为了实现多线程安全，在几乎所有的方法上都加上了synchronized锁，而加锁的结果就是HashTable操作的效率十分低下。

## HashTable与HashMap对比

（1）线程安全：HashMap是线程不安全的类，多线程下会造成并发冲突，但单线程下运行效率较高；HashTable是线程安全的类，很多方法都是用synchronized修饰，但同时因为加锁导致并发效率低下，单线程环境效率也十分低；

（2）插入null：HashMap允许有一个键为null，允许多个值为null；但HashTable不允许键或值为null；

（3）容量：HashMap底层数组长度必须为2的幂，这样做是为了hash准备，默认为16；而HashTable底层数组长度可以为任意值，这就造成了hash算法散射不均匀，容易造成hash冲突，默认为11；

（4）Hash映射：HashMap的hash算法通过非常规设计，将底层table长度设计为2的幂，使用位与运算代替取模运算，减少运算消耗；而HashTable的hash算法首先使得hash值小于整型数最大值，再通过取模进行散射运算；

```java
// HashMap
static final int hash(Object key) {
    int h;
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
// 下标index运算
int index = (table.length - 1) & hash(key)

// HashTable
int hash = key.hashCode();
int index = (hash & 0x7FFFFFFF) % tab.length;
```

（5）扩容机制：HashMap创建一个为原先2倍的数组，然后对原数组进行遍历以及rehash；HashTable扩容将创建一个原长度2倍的数组，再使用头插法将链表进行反序；

（6）结构区别：HashMap是由数组+链表形成，在JDK1.8之后链表长度大于8时转化为红黑树；而HashTable一直都是数组+链表；

（7）继承关系：HashTable继承自Dictionary类；而HashMap继承自AbstractMap类；

（8）迭代器：HashMap是fail-fast（查看之前HashMap相关文章）；而HashTable不是。
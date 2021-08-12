# ArrayList底层原理源码分析

# 1 简介

ArrayList是用数组实现的，并且它是动态数组，也就是它的容量是可以自动增长的，看下面的类声明可知道它实现了众多接口，比如List，RandomAccess，Serializable，Cloneable



```java
public class ArrayList<E> extends AbstractList<E>
        implements List<E>, RandomAccess, Cloneable, java.io.Serializable
```

- 实现RandomAccess接口：所以ArrayList支持快速随机访问，本质上是通过下标序号随机访问
- 实现Serializable接口：使ArrayList支持序列化，通过序列化传输
- 实现Cloneable接口：使ArrayList能够克隆

# 2 底层关键



```css
transient Object[] elementData;
```

ArrayList底层本质上是一个数组，用该数组来保存数据

`transient`:Java关键字，变量修饰符，如果用transient声明一个实例变量，当对象存储时，它的值不需要维持。换句话来说就是，用transient关键字标记的成员变量不参与序列化过程。

> Java的[serialization](https://link.jianshu.com?t=https://baike.baidu.com/item/serialization)提供了一种持久化对象实例的机制。当持久化对象时，可能有一个特殊的对象数据成员，我们不想用serialization机制来保存它。为了在一个特定对象的一个域上关闭serialization，可以在这个域前加上关键字transient。当一个对象被序列化的时候，transient型变量的值不包括在序列化的表示中，然而非transient型的变量是被包括进去的。

# 3 构造函数

-  `ArrayList()`：默认构造方法，构造一个初始容量为0的列表



```cpp
    public ArrayList() {
        //无参构造器默认构造一个空数组
        this.elementData = DEFAULTCAPACITY_EMPTY_ELEMENTDATA;
    }
```

-  `ArrayList(int initialCapacity)`：构造一个具有指定初始容量的空列表



```cpp
  public ArrayList(int initialCapacity) {
        if (initialCapacity > 0) {
            //如果初始化容量大于零，则新建一个数组容量为initialCapacity
            this.elementData = new Object[initialCapacity];
        } else if (initialCapacity == 0) {
             //如果初始化容量等于零，则新建一个数组容量为空
            this.elementData = EMPTY_ELEMENTDATA;
        } else {
             //小于零时报异常
            throw new IllegalArgumentException("Illegal Capacity: "+
                                               initialCapacity);
        }
    }
```

-  `ArrayList(Collection c)`:创建一个包含collection的ArrayList，返回的元素是按照该 collection 的迭代器在返回它们的顺序排列的（具体是在Collection的toArray()方法中）



```kotlin
   //创建一个包含collection的ArrayList 
    public ArrayList(Collection<? extends E> c) {
        //返回包含此 collection 中所有元素的数组，元素是按照该 collection 的迭代器返回它们的顺序排列的
        elementData = c.toArray();
        if ((size = elementData.length) != 0) {
            // c.toArray might (incorrectly) not return Object[] (see 6260652)
           //如果数组长度不等于0（就是有数据）
            if (elementData.getClass() != Object[].class)
                //复制elementData数组，截取或用 null 填充（如有必要），以使副本具有指定的长度，并返回Object[]类
                elementData = Arrays.copyOf(elementData, size, Object[].class);
        } else {
            // replace with empty array.
            this.elementData = EMPTY_ELEMENTDATA;
        }
    }
```

# 4 工具类

-  `trimToSize()`: 将当前容量值设为实际元素个数



```cpp
 public void trimToSize() {
        modCount++;
        if (size < elementData.length) {
            //相当于缩小了容量
            elementData = Arrays.copyOf(elementData, size);
        }
    }
```

-  `public void ensureCapacity(int minCapacity)`:扩容方法，ArrayList每次新增元素都会进行容量大小检测判断，若新增的后元素的个数会超过ArrayList的容量，就会进行扩容满足新增元素的需求



```cpp
public void ensureCapacity(int minCapacity) {
        int minExpand = (elementData != EMPTY_ELEMENTDATA)
            ? 0
            //默认构造情况下，elementData 是空的，所以minExpand为10
            : DEFAULT_CAPACITY;

        if (minCapacity > minExpand) {
            ensureExplicitCapacity(minCapacity);
        }
    }
```

接下来看`ensureExplicitCapacity()`，



```cpp
private void ensureExplicitCapacity(int minCapacity) {
    // 数据结构发生改变，和fail-fast机制有关，在使用迭代器过程中，只能通过迭代器的方法（比如迭代器中add，remove等），修改List的数据结构，
    // 如果使用List的方法（比如List中的add，remove等），修改List的数据结构，会抛出ConcurrentModificationException
        modCount++;

        if (minCapacity - elementData.length > 0)
           //当前数组容量大小不足时，才会调用grow方法，自动扩容
            grow(minCapacity);
    }
```

> [fail-fast机制](https://link.jianshu.com?t=http://blog.csdn.net/chenssy/article/details/38151189)

实际上扩容的方法grow()



```cpp
    private void grow(int minCapacity) {
        // overflow-conscious code
        int oldCapacity = elementData.length;
        // 新的容量大小 = 原容量大小的1.5倍，右移1位并相加本身近似于1.5倍
        int newCapacity = oldCapacity + (oldCapacity >> 1);
        if (newCapacity - minCapacity < 0)
             //溢出判断
            newCapacity = minCapacity;
        if (newCapacity - MAX_ARRAY_SIZE > 0)
            newCapacity = hugeCapacity(minCapacity);
        // minCapacity is usually close to size, so this is a win:
        elementData = Arrays.copyOf(elementData, newCapacity);
    }
```

1.5倍的神秘规律：因为一次性扩容太大(例如2.5倍)可能会浪费更多的内存
 1.5倍：最多浪费33%
 2.5倍：最多会浪费60%
 3.5倍：则会浪费71%
 但是一次性扩容太小，需要多次对数组重新分配内存，对性能消耗比较严重。**所以1.5倍刚刚好**，既能满足性能需求，也不会造成很大的内存消耗。
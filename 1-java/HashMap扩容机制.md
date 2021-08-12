java 1.7



java 1.8

1，在resize()方法中，定义了oldCap参数，记录了原table的长度，定义了newCap参数，记录新table长度，newCap是oldCap长度的2倍（注释1），同时扩展点也乘2。

2，注释2是循环原table，把原table中的每个链表中的每个元素放入新table。

3，注释3，e.next==null，指的是链表中只有一个元素，所以直接把e放入新table，其中的e.hash & (newCap - 1)就是计算e在新table中的位置，和JDK1.7中的indexFor()方法是一回事。

4，注释// preserve order，这个注释是源码自带的，这里定义了4个变量：loHead，loTail，hiHead，hiTail，看起来可能有点眼晕，其实这里体现了JDK1.8对于计算节点在table中下标的新思路：

正常情况下，计算节点在table中的下标的方法是：hash&(oldTable.length-1)，扩容之后，table长度翻倍，计算table下标的方法是hash&(newTable.length-1)，也就是hash&(oldTable.length*2-1)，于是我们有了这样的结论：这新旧两次计算下标的结果，要不然就相同，要不然就是新下标等于旧下标加上旧数组的长度。

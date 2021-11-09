# Java双端队列Deque使用详解

Deque是一个双端队列接口，继承自Queue接口，Deque的实现类是LinkedList、ArrayDeque、LinkedBlockingDeque，其中LinkedList是最常用的。

## Queue

Queue的6个方法分类：

压入元素(添加)：add()、offer()
相同：未超出容量，从队尾压入元素，返回压入的那个元素。
区别：在超出容量时，add()方法会对抛出异常，offer()返回false

弹出元素(删除)：remove()、poll()
相同：容量大于0的时候，删除并返回队头被删除的那个元素。
区别：在容量为0的时候，remove()会抛出异常，poll()返回false

获取队头元素(不删除)：element()、peek()
相同：容量大于0的时候，都返回队头元素。但是不删除。
区别：容量为0的时候，element()会抛出异常，peek()返回null。

## Deque

Deque有三种用途：
普通队列(一端进另一端出):
Queue queue = new LinkedList()或Deque deque = new LinkedList()
双端队列(两端都可进出)
Deque deque = new LinkedList()
堆栈
Deque deque = new LinkedList()
注意：Java堆栈Stack类已经过时，Java官方推荐使用Deque替代Stack使用。Deque堆栈操作方法：push()、pop()、peek()。

Deque是一个线性collection，支持在两端插入和移除元素。名称 deque 是“double ended queue（双端队列）”的缩写，通常读为“deck”。大多数 Deque 实现对于它们能够包含的元素数没有固定限制，但此接口既支持有容量限制的双端队列，也支持没有固定大小限制的双端队列。

此接口定义在双端队列两端访问元素的方法。提供插入、移除和检查元素的方法。每种方法都存在两种形式：一种形式在操作失败时抛出异常，另一种形式返回一个特殊值（null 或 false，具体取决于操作）。插入操作的后一种形式是专为使用有容量限制的 Deque 实现设计的；在大多数实现中，插入操作不能失败。

下表总结了上述 12 种方法：

|      | 第一个元素 (头部) |               |              | 最后一个元素 (尾部) |
| ---- | ----------------- | ------------- | ------------ | ------------------- |
|      | 抛出异常          | 特殊值        | 抛出异常     | 特殊值              |
| 插入 | addFirst(e)       | offerFirst(e) | addLast(e)   | offerLast(e)        |
| 删除 | removeFirst()     | pollFirst()   | removeLast() | pollLast()          |
| 检查 | getFirst()        | peekFirst()   | getLast()    | peekLast()          |


Deque接口扩展(继承)了 Queue 接口。在将双端队列用作队列时，将得到 FIFO（先进先出）行为。将元素添加到双端队列的末尾，从双端队列的开头移除元素。从 Queue 接口继承的方法完全等效于 Deque 方法，如下表所示：

| Queue方法 | 等效Deque方法 |
| --------- | ------------- |
| add(e)    | addLast(e)    |
| offer(e)  | offerLast(e)  |
| remove()  | removeFirst() |
| poll()    | pollFirst()   |
| element() | getFirst()    |
| peek()    | peekFirst()   |


双端队列也可用作 LIFO（后进先出）堆栈。应优先使用此接口而不是遗留 Stack 类。在将双端队列用作堆栈时，元素被推入双端队列的开头并从双端队列开头弹出。堆栈方法完全等效于 Deque 方法，如下表所示：

| 堆栈方法 | 等效Deque方法 |
| -------- | ------------- |
| push(e)  | addFirst(e)   |
| pop()    | removeFirst() |
| peek()   | peekFirst()   |



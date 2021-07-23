# LinkedList底层原理

![](https://img-blog.csdn.net/20160412130036633?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

如图所示 LinkedList 底层是基于双向链表实现的，也是实现了 List 接口，所以也拥有 List 的一些特点(JDK1.7/8 之后取消了循环，修改为双向链表)。

新增方法

 可见每次插入都是移动指针，和 ArrayList 的拷贝数组来说效率要高上不少。

查询方法

    public E get(int index) {
        checkElementIndex(index);
        return node(index).item;
    }
    
    Node<E> node(int index) {
        // assert isElementIndex(index);
     
        if (index < (size >> 1)) {
            Node<E> x = first;
            for (int i = 0; i < index; i++)
                x = x.next;
            return x;
        } else {
            Node<E> x = last;
            for (int i = size - 1; i > index; i--)
                x = x.prev;
            return x;
        }
    }

由此可以看出是使用二分查找来看 index 离 size 中间距离来判断是从头结点正序查还是从尾节点倒序查。

node()会以O(n/2)的性能去获取一个结点 如果索引值大于链表大小的一半，那么将从尾结点开始遍历
这样的效率是非常低的，特别是当 index 越接近 size 的中间值时。

一开始我很费解，这是要干嘛？后来我才明白，代码要做的是:判断给定的索引值，若索引值大于整个链表长度的一半，则从后往前找，若索引值小于整个链表的长度的一半，则从前往后找。这样就可以保证，不管链表长度有多大，搜索的时候最多只搜索链表长度的一半就可以找到，大大提升了效率。

总结：

LinkedList 插入，删除都是移动指针效率很高。
查找需要进行遍历查询，效率较低。

![](https://img-blog.csdn.net/20160412130036633?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

对比一下，知道区别在哪里了吧？在1.7中，去掉了环形结构，自然在代码中的也会有部分的改变。
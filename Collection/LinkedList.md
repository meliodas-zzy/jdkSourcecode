### LinkedList

`LinkedList`既实现了`List`，又实现了`Deque`，前者使它能够像使用`ArrayList`一样使用，后者又使它能够承担队列的职责。`LinkedList`内部结构是一个双向链表

```
transient int size = 0;
transient Node<E> first;
transient Node<E> last;

private static class Node<E> {
    E item;
    Node<E> next;
    Node<E> prev;

    Node(Node<E> prev, E element, Node<E> next) {
        this.item = element;
        this.next = next;
        this.prev = prev;
    }
}
```

`LinkedList`非常适合大量数据的插入与删除，但其对处于中间位置的元素，无论是增删还是改查都需要折半遍历，这在数据量大时会十分影响性能。

**折半遍历**

```
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
```

实现了List和Deque的方法

![image-20220821120649098](C:\Users\zhanyu\AppData\Roaming\Typora\typora-user-images\image-20220821120649098.png)
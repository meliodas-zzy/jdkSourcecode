#### Deque

![image-20220821113843426](C:\Users\zhanyu\AppData\Roaming\Typora\typora-user-images\image-20220821113843426.png)

双端队列接口

```java
//在队首添加元素
void addFirst(E e);
//在队首添加元素
boolean offerFirst(E e);

//在队尾添加元素
void addLast(E e);
boolean offerLast(E e);

//删除队首元素
E removeFirst();
E pollFirst();

//删除队尾元素
E removeLast();
E pollLast();

//获取队首元素
E getFirst();
E peekFirst();

//获取队尾元素
E getLast();
E peekLast();

//删除第一个事件，大多数指的是删除第一个和 o equals的元素
boolean removeFirstOccurrence(Object o);
//删除最后一个事件，大多数指的是删除最后一个和 o equals的元素
boolean removeLastOccurrence(Object o);
```

```java
//与addLast(E e)等价
boolean add(E e);

//与offerLast(E e)等价
boolean offer(E e);

//与removeFirst()等价
E remove();

//与pollFirst()等价
E poll();

//与getFirst()等价
E element();

//与peekFirst()等价
E peek();
```

用于实现Stack的方法定义，仅在队首执行插入删除

```cpp
//与addFirst()等价
void push(E e);

//与removeFirst()等价
E pop();
```

继承自Collections的方法

```csharp
//顺序是从队首到队尾
Iterator<E> iterator();

//顺序是从队尾到队首
Iterator<E> descendingIterator();
```

#### ArrayDeque

`ArrayDeque`通过循环数组的方式实现的循环队列，并通过位运算来提高效率，容量大小始终是2的次幂。当数据充满数组时，它的容量将翻倍。

#### LinkedList

LinkedList即实现了List又实现了Deque，通过链表方式实现双端队列相关功能
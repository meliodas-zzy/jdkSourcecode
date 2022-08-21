### Iterator

1、迭代器模式封装集合内部的复杂数据结构，开发者不需要了解如何遍历，直接使用容器提供的迭代器即可；

2、迭代器模式将集合对象的遍历操作从集合类中拆分出来，放到迭代器类中，让两者的职责更加单一；

3、迭代器模式让添加新的遍历算法更加容易，更符合开闭原则。除此之外，因为迭代器都实现自相同的接口，在开发中，基于接口而非实现编程，替换迭代器也变得更加容易。

java的Iterator接口定义了四个方法，Iterator内部由一个cursor记录游标

![image-20220821100831203](C:\Users\zhanyu\AppData\Roaming\Typora\typora-user-images\image-20220821100831203.png)

集合的具体实现类会在内部组合一个迭代器对象，并且可依据和多种不同的呆呆其类型，以支持不同的遍历方法，提供多种迭代器功能，以ArrayList为例

![image-20220821101025240](C:\Users\zhanyu\AppData\Roaming\Typora\typora-user-images\image-20220821101025240.png)

其中Itr提供基本的next,hashNext操作

![image-20220821101123851](C:\Users\zhanyu\AppData\Roaming\Typora\typora-user-images\image-20220821101123851.png)

而ListItr提供了更多功能

![image-20220821101209854](C:\Users\zhanyu\AppData\Roaming\Typora\typora-user-images\image-20220821101209854.png)



##### fail-fast

在使用迭代器进行遍历时，若对集合进行可增删改操作，则有可能对遍历结果产生影响，比如对于集合{a,b,c,d}遍历，当前遍历到b，此时删除a，那么b,c,d将向前移一位，迭代器此时指向c，遍历结果中将不包含b，因此遍历出错。因此设计了fail-fast机制，集合会维持一个modCount变量记录更改次数，在初始化Iterator时，会将当前的modCount赋值给迭代器对象的ecpectedModConut属性，之后迭代器每次操作首先执行检查操作，确定是否有修改发生，有则直接返回异常

```
public E next() {
    checkForComodification();
    int i = cursor;
    if (i >= size)
        throw new NoSuchElementException();
    Object[] elementData = ArrayList.this.elementData;
    if (i >= elementData.length)
        throw new ConcurrentModificationException();
    cursor = i + 1;
    return (E) elementData[lastRet = i];
}
```

```
final void checkForComodification() {
    if (modCount != expectedModCount)
        throw new ConcurrentModificationException();
}
```

##### 支持增删改的迭代器

有的迭代器支持增删改操作，比如ListItr的set(), add()操作，实现原理为修改cursor及lastRet值（游标之前的那个坐标）

```
public void set(E e) {
    if (lastRet < 0)
        throw new IllegalStateException();
    checkForComodification();

    try {
        ArrayList.this.set(lastRet, e);
    } catch (IndexOutOfBoundsException ex) {
        throw new ConcurrentModificationException();
    }
}
```

##### Iterable

作用是为实现这个接口的类提供for-each操作，主要方法是返回一个迭代器对象，因此for-each操作底层原理是迭代器机制
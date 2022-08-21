#### Collection接口

```
集合层次结构中的根接口。集合表示一组对象，称为其元素。一些集合允许重复元素，而另一些则不允许。有些是有序的，有些是无序的。 JDK 不提供此接口的任何直接实现：它提供更具体的子接口（如 Set 和 List）的实现。
```

三大子接口：List, Set, Queue

##### List

```
有序集合（也称为序列）。用户可以精确控制每个元素在列表中的插入位置。用户可以通过整数索引（列表中的位置）访问元素，并在列表中搜索元素。
```

关注一下AbstractList的equals和hashCode方法

```
//首先判断是否为同一元素，然后每一位按值比较
public boolean equals(Object o) {
    if (o == this)
        return true;
    if (!(o instanceof List))
        return false;

    ListIterator<E> e1 = listIterator();
    ListIterator<?> e2 = ((List<?>) o).listIterator();
    while (e1.hasNext() && e2.hasNext()) {
        E o1 = e1.next();
        Object o2 = e2.next();
        if (!(o1==null ? o2==null : o1.equals(o2)))
            return false;
    }
    return !(e1.hasNext() || e2.hasNext());
}
```

```
//按照每一位计算hash值
public int hashCode() {
    int hashCode = 1;
    for (E e : this)
        hashCode = 31*hashCode + (e==null ? 0 : e.hashCode());
    return hashCode;
}
```

##### Set

直接来看HashSet实现，很简单，基于HashMap实现

成员变量

```
//存储数据
private transient HashMap<E,Object> map;

// Dummy value to associate with an Object in the backing Map
//代替value值
private static final Object PRESENT = new Object();
```

add操作

```
public boolean add(E e) {
	//map.put(k,v)将返回previousK，即如果这个k已存在则返回之前的k值，不存在返回null
	//所以在set中，k值之前已存在则再次添加返回false，不存在则返回true
    return map.put(e, PRESENT)==null;
}
```

remove操作同样是使用HashMap的方法

```
public boolean remove(Object o) {
    return map.remove(o)==PRESENT;
}
```

同理LinedHashSet则是复用的LinkedHashSet的方法即数据结构

##### Queue

```
//添加成功但会true，失败返回false，超限抛出异常
boolean add(E e);
//与add相比，超过容量上限时不会抛出异常，而是返回false
boolean offer(E e);
//删除并返回队首元素，队列为空抛出异常
E remove();
//与remove相比，队列为空时返回null
E poll();
//返回队首元素，但不删除，若队列为空抛出异常
E element();
//与element相比，队列为空时返回null
E peek();
```


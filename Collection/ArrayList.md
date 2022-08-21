### ArrayList

继承结构

![image-20220821110443755](C:\Users\zhanyu\AppData\Roaming\Typora\typora-user-images\image-20220821110443755.png)

其中Cloneable, RandomAccess, Serializable都仅作为标识表示实现类具备某项功能。`RandomAccess`表示实现类支持快速随机访问，`Cloneable`表示实现类支持克隆，具体表现为重写了`clone`方法，`java.io.Serializable`则表示支持序列化。

#### 主要属性

```
private static final int DEFAULT_CAPACITY = 10;

private static final Object[] EMPTY_ELEMENTDATA = {};

private static final Object[] DEFAULTCAPACITY_EMPTY_ELEMENTDATA = {};
//存储元素
transient Object[] elementData; // non-private to simplify nested class access

private int size;
```

#### 初始大小

默认构造方法初始大小为0

```
private static final Object[] DEFAULTCAPACITY_EMPTY_ELEMENTDATA = {};
public ArrayList() {
    this.elementData = DEFAULTCAPACITY_EMPTY_ELEMENTDATA;
}
```

#### CRUD

**查**

```
private void rangeCheck(int index) {
    if (index >= size)
        throw new IndexOutOfBoundsException(outOfBoundsMsg(index));
}
```

```
public E get(int index) {
    rangeCheck(index);

    return elementData(index);
}
```

**改**

```
public E set(int index, E element) {
    rangeCheck(index);

    E oldValue = elementData(index);
    elementData[index] = element;
    return oldValue;
}
```

**增**

```
public boolean add(E e) {
	//检查容量是否够用，不够用则需要扩容，这个方法会导致modCount增加
    ensureCapacityInternal(size + 1);  // Increments modCount!!
    elementData[size++] = e;
    return true;
}
private void ensureCapacityInternal(int minCapacity) {
        ensureExplicitCapacity(calculateCapacity(elementData, minCapacity));
}
//这里可以看出在首次add操作时，回将空数组初始化为DEFAULT_CAPICITY，即10的大小
private static int calculateCapacity(Object[] elementData, int minCapacity) {
        if (elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA) {
            return Math.max(DEFAULT_CAPACITY, minCapacity);
        }
        return minCapacity;
    }
```

ensureCapacityInternal()最终调用下边这个方法

```
private void ensureExplicitCapacity(int minCapacity) {
    modCount++;

    // overflow-conscious code
    if (minCapacity - elementData.length > 0)
        grow(minCapacity);
}
```

```
private void grow(int minCapacity) {
    // overflow-conscious code
    int oldCapacity = elementData.length;
    //扩充至1.5倍
    int newCapacity = oldCapacity + (oldCapacity >> 1);
    //如果还不满足所需的最小容量，则扩充至所需最小容量
    if (newCapacity - minCapacity < 0)
        newCapacity = minCapacity;
    //溢出检查
    if (newCapacity - MAX_ARRAY_SIZE > 0)
        newCapacity = hugeCapacity(minCapacity);
    // minCapacity is usually close to size, so this is a win:
    elementData = Arrays.copyOf(elementData, newCapacity);
}
```

总结一下扩容机制为：

首先创建一个空数组**elementData**，第一次插入数据时直接扩充至10，然后如果**elementData**的长度不足，就扩充1.5倍，如果扩充完还不够，就使用需要的长度作为**elementData**的长度。

**删**

```
public E remove(int index) {
    rangeCheck(index);

    modCount++;
    E oldValue = elementData(index);

    int numMoved = size - index - 1;
    if (numMoved > 0)
        System.arraycopy(elementData, index+1, elementData, index,
                         numMoved);
    elementData[--size] = null; // clear to let GC do its work

    return oldValue;
}
```

删除元素，移动元素，修改size
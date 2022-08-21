### Map接口

#### Entry

存储在Map中的数据需要实现此接口，主要提供对key和value的操作，也是我们使用最多的操作。

```java
// 获取对应的key
K getKey();

// 获取对应的value
V getValue();

// 替换原有的value
V setValue(V value);

// 希望我们实现equals和hashCode
boolean equals(Object o);
int hashCode();

// 从1.8起，还提供了比较的方法，类似的方法共四个
public static <K extends Comparable<? super K>, V> Comparator<Map.Entry<K,V>> comparingByKey() {
        return (Comparator<Map.Entry<K, V>> & Serializable)
            (c1, c2) -> c1.getKey().compareTo(c2.getKey());
}
```

#### Map重要方法

```java
// 返回当前数据个数
int size();

// 是否为空
boolean isEmpty();

// 判断是否包含key，这里用到了key的equals方法，所以key必须实现它
boolean containsKey(Object key);

// 判断是否有key保存的值是value，这也基于equals方法
boolean containsValue(Object value);

// 通过key获取对应的value值
V get(Object key);

// 存入key-value
V put(K key, V value);

// 移除一个key-value对
V remove(Object key);

// 从其他Map添加
void putAll(Map<? extends K, ? extends V> m);

// 清空
void clear();

// 返回所有的key至Set集合中，因为key是不可重的，Set也是不可重的
Set<K> keySet();

// 返回所有的values
Collection<V> values();

// 返回key-value对到Set中
Set<Map.Entry<K, V>> entrySet();

// 希望我们实现equals和hashCode
boolean equals(Object o);
int hashCode();

default V getOrDefault(Object key, V defaultValue) {
    V v;
    return (((v = get(key)) != null) || containsKey(key))
        ? v
        : defaultValue;
}
```

#### SortedMap接口

`SortedMap`接口为有序数据服务的。

`SortedMap`接口需要数据的key支持`Comparable`，或者可以被指定的`Comparator`接受。`SortedMap`主要提供了以下方法：

```java
// 返回排序数据所用的Comparator
Comparator<? super K> comparator();

// 返回在[fromKey, toKey)之间的数据
SortedMap<K,V> subMap(K fromKey, K toKey);

// 返回从第一个元素到toKey之间的数据
SortedMap<K,V> headMap(K toKey);

// 返回从fromKey到末尾之间的数据
SortedMap<K,V> tailMap(K fromKey);

//返回第一个数据的key
K firstKey();

//返回最后一个数据的key
K lastKey();
```

#### Navigable接口

SortedMap的子接口，对于一个已经排序的数据集，除了最大值与最小值之外，我们可以对任何一个元素，找到比它小的值和比它大的值，还可以按照按照原有的顺序倒序排序等。`NavigableMap`就为我们提供了这些功能。

```csharp
// 找到第一个比指定的key小的值
Map.Entry<K,V> lowerEntry(K key);

// 找到第一个比指定的key小的key
K lowerKey(K key);

// 找到第一个小于或等于指定key的值
Map.Entry<K,V> floorEntry(K key);

// 找到第一个小于或等于指定key的key
K floorKey(K key);

//  找到第一个大于或等于指定key的值
Map.Entry<K,V> ceilingEntry(K key);

K ceilingKey(K key);

// 找到第一个大于指定key的值
Map.Entry<K,V> higherEntry(K key);

K higherKey(K key);

// 获取最小值
Map.Entry<K,V> firstEntry();

// 获取最大值
Map.Entry<K,V> lastEntry();

// 删除最小的元素
Map.Entry<K,V> pollFirstEntry();

// 删除最大的元素
Map.Entry<K,V> pollLastEntry();

//返回一个倒序的Map
NavigableMap<K,V> descendingMap();

// 返回一个Navigable的key的集合，NavigableSet和NavigableMap类似
NavigableSet<K> navigableKeySet();

// 对上述集合倒序
NavigableSet<K> descendingKeySet();
```

#### TreeMap

实现了Navigable接口，实现了红黑树，**红黑树**能保证增、删、查等基本操作的时间复杂度为**O(lgN)**
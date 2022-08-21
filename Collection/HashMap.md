### HashMap

存储结构

```
//普通节点
static class Node<K,V> implements Map.Entry<K,V> {
    final int hash;
    final K key;
    V value;
    Node<K,V> next;

    ...
}
```

```
//红黑树节点
static final class TreeNode<K,V> extends LinkedHashMap.Entry<K,V> {
    TreeNode<K,V> parent;  // red-black tree links
    TreeNode<K,V> left;
    TreeNode<K,V> right;
    TreeNode<K,V> prev;    // needed to unlink next upon deletion
    boolean red;
	...
}
```

成员变量

```
//HashMap的大大小永远都是2 ^ n,自定义大小时，程序会自动设置为离自定义数值最接近的2 ^ n
static final int DEFAULT_INITIAL_CAPACITY = 1 << 4; // aka 16

static final int MAXIMUM_CAPACITY = 1 << 30;

static final float DEFAULT_LOAD_FACTOR = 0.75f;

static final int TREEIFY_THRESHOLD = 8;
//树大小为6，还会转换为链表
static final int UNTREEIFY_THRESHOLD = 6;

static final int MIN_TREEIFY_CAPACITY = 64;

transient Node<K,V>[] table;
//提供遍历的功能
transient Set<Map.Entry<K,V>> entrySet;

transient int size;

transient int modCount;
//下一次扩容值
int threshold;

final float loadFactor;
```

#### put

```
public V put(K key, V value) {
    return putVal(hash(key), key, value, false, true);
}
static final int hash(Object key) {
    int h;
    //key的hash值与其高16为相与
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
```

```
final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
               boolean evict) {
    Node<K,V>[] tab; Node<K,V> p; int n, i;
    if ((tab = table) == null || (n = tab.length) == 0)
        n = (tab = resize()).length;
    //n = 2 ^ n 时,hash % n == (n - 1) & hash将取余操作转换为位操作，加快运算速度
    if ((p = tab[i = (n - 1) & hash]) == null)
        tab[i] = newNode(hash, key, value, null);
    else {
        Node<K,V> e; K k;
        if (p.hash == hash &&
            ((k = p.key) == key || (key != null && key.equals(k))))
            e = p;
        else if (p instanceof TreeNode)
            e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
        else {
            for (int binCount = 0; ; ++binCount) {
                if ((e = p.next) == null) {
                    p.next = newNode(hash, key, value, null);
                    if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                        treeifyBin(tab, hash);
                    break;
                }
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    break;
                p = e;
            }
        }
        if (e != null) { // existing mapping for key
            V oldValue = e.value;
            if (!onlyIfAbsent || oldValue == null)
                e.value = value;
            afterNodeAccess(e);
            return oldValue;
        }
    }
    ++modCount;
    if (++size > threshold)
        resize();
    afterNodeInsertion(evict);
    return null;
}
```

总结

hashmap使用Node存储节点信息，使用拉链法解决hash冲突，担当拉链长度大于8时，会进行树化，树化treefyBin函数执行树化逻辑，将链表转换成红黑树前会判断，如果当前数组的长度小于 64，那么会选择先进行数组扩容，而不是转换为红黑树。若大于64才真正进行转换，Node节点也会转换为TreeNode节点。

#### LinkedHashMap

继承了HashMap，**LinkedHashMap保证了元素迭代的顺序**。**该迭代顺序可以是插入顺序或者是访问顺序。**由参数accessOrder决定，默认为false，会维持插入顺序，为true时会维持访问顺序，可以用于实现LRUCache。没有自己实现增删改查的方法，复用了hashmap中的方法，自己进行了必要的修改，比如在get方法中

```
public V get(Object key) {
    Node<K,V> e;
    if ((e = getNode(hash(key), key)) == null)
        return null;
    if (accessOrder)
        afterNodeAccess(e);
    return e.value;
}
```

相较于hashmap，多了afterNodeAccess（）方法，将最近访问的节点移至链表末尾
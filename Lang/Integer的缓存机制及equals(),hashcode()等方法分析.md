**Integer简介**

Integer为int基本数据类型的包装器类型。可以通过intValue()获取包装器类型对应的int值，

可以通过valueOf(int)将一个int值转换为Integer对象。

Integer同样改写了Object中的equals()和hashcode()方法，也是为了实现对值的比较。

Integer具有缓存机制，可以快速获取-128~127范围内的Integer对象。

**intValue()与valueOf():**

Integer内部拥有成员变量value，用于保存int值

```
private final int value;
```

intValue()方法就是返回这个value值：

```
public int intValue() {
        return value;
    }
```

valueOf()用于将传入的值转换为Integer对象

```
public static Integer valueOf(int i) {
    if (i >= IntegerCache.low && i <= IntegerCache.high)
    	//若i在缓存范围内，直接返回缓存中的对象
        return IntegerCache.cache[i + (-IntegerCache.low)];
    return new Integer(i);
}
```

Integer通过一个静态内部类实现缓存机制

```
private static class IntegerCache {
    static final int low = -128;
    static final int high;
    static final Integer cache[];

    static {
        // 缓存最大值是可以配置的，默认为127
        int h = 127;
        String integerCacheHighPropValue =
            sun.misc.VM.getSavedProperty("java.lang.Integer.IntegerCache.high");
        //获取一个integerCacheHighPropValue，并选取127与integerCacheHighPropValue中较大的那个作为high的值
        if (integerCacheHighPropValue != null) {
            try {
                int i = parseInt(integerCacheHighPropValue);
                i = Math.max(i, 127);
                // Maximum array size is Integer.MAX_VALUE
                h = Math.min(i, Integer.MAX_VALUE - (-low) -1);
            } catch( NumberFormatException nfe) {
                // If the property cannot be parsed into an int, ignore it.
            }
        }
        high = h;
		
		//为缓存数组赋值
        cache = new Integer[(high - low) + 1];
        int j = low;
        for(int k = 0; k < cache.length; k++)
            cache[k] = new Integer(j++);

        // range [-128, 127] must be interned (JLS7 5.1.7)
        assert IntegerCache.high >= 127;
    }

    private IntegerCache() {}
}
```

**equals()与hashcode()方法：**

equals()源码如下：

```
public boolean equals(Object obj) {
    if (obj instanceof Integer) {
        return value == ((Integer)obj).intValue();
    }
    return false;
}
```

可以看出比较的值。

而hashcode()更为直接：

```
public static int hashCode(int value) {
    return value;
}
```

直接返回value。

因此Integer也实现了值的比较。
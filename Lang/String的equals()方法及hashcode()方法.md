**Object:**

java中所有的类都继承自object类，object中定义了wait()/notify(), equals(), hashcode(), toString(), clone(), finalize()等方法。

**== 运算符与Object中的equals():**

由于java是值传递的，因此==运算符在比较基本数据类型时，比较的是基本数据类型的值是否相等；在比较引用数据类型时，比较的是他们指向的地址是否相同，即两个引用是否指向同一对象。

Object中的equals()方法源码为

```
public boolean equals(Object obj) {
        return (this == obj);
}
```

内部使用==进行比较，且接受的参数为引用数据类型，可见equals()方法用于引用数据类型的比较，且在不重写的情况下比较的是内存地址，即是否指向同一对象。

**String中的equals()：**

String重写了Object的equals()方法，源码如下：

```
public boolean equals(Object anObject) {
        if (this == anObject) {
            return true;
        }
        if (anObject instanceof String) {
            String anotherString = (String)anObject;
            int n = value.length;
            if (n == anotherString.value.length) {
                char v1[] = value;
                char v2[] = anotherString.value;
                int i = 0;
                while (n-- != 0) {
                    if (v1[i] != v2[i])
                        return false;
                    i++;
                }
                return true;
            }
        }
        return false;
    }
```

执行逻辑为：

首先判断是否为同一个对象，若是直接返回true，若不是则继续执行下述逻辑；

判断入参是否为String对象，若不是，直接返回false，若相等继续执行下述逻辑；

判断入参长度是否与this的长度相等，若不相等直接返回false，若相等继续执行下述逻辑；

从尾部开始遍历每个字符，比较是否相等，在此过程中若有字符不相等，直接返回false；

若全部字符均相等，则返回true。

从执行逻辑可以看出String通过改写equals方法实现了引用数据类型所指向的对象的值的比较，而不是默认的比较两个对象是否为同一对象。

**String中的hashcode():**

String同样重写了hashcode()方法，实际上由于java规定若equals()判断相等的对象，其hashcode必须相等，因此重写了equals(),就必须要重写hashcode()，否则会出现equals判断相等，但其hahs值反而不相等的情况。String的hashcode()源码如下：

```
public int hashCode() {
        int h = hash;
        if (h == 0 && value.length > 0) {
            char val[] = value;

            for (int i = 0; i < value.length; i++) {
                h = 31 * h + val[i];
            }
            hash = h;
        }
        return h;
    }
```

从h = 31 * h + val[i];中可以看出此方法根据String中的每一位字符产生hash值，而Object中的hashcode()方法调用的是Native方法，是按照内存地址计算的hash值。
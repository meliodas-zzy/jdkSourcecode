**ThreadLocal简介：**

threadlocal可以为线程提供线程本地变量，这个数据绑定在当前线程中的，只有当前线程可以访问。

要获取线程本地变量，只需要使用threadlocal.get()方法，要设置本地变量值，只需使用threadlocal,set()方法。

**ThreadLocal.get():**

首先来看get():

```
public T get() {
	//获取当前线程
    Thread t = Thread.currentThread();
    //获取当前线程中的ThreadLocalMap
    ThreadLocalMap map = getMap(t);
    if (map != null) {
    	//ThreadLocalMap中节点为Entry
        ThreadLocalMap.Entry e = map.getEntry(this);
        if (e != null) {
            @SuppressWarnings("unchecked")
            //获取本地变量
            T result = (T)e.value;
            return result;
        }
    }
    return setInitialValue();
}
```

get()首先会获取ThreadLocalMap，这个Map实际存储在当前线程中，源码如下：

```
ThreadLocalMap getMap(Thread t) {
    return t.threadLocals;
}
```

t中存放着这个map

```
public class Thread implements Runnable {
	...
	ThreadLocal.ThreadLocalMap threadLocals = null;
	...
}
```

获取到ThreadLocalMap后，会根据当前threadlocal获取ThreadLocalMap中存储的Entry,Entry结构如下：

```
static class Entry extends WeakReference<ThreadLocal<?>> {
    /** The value associated with this ThreadLocal. */
    Object value;

    Entry(ThreadLocal<?> k, Object v) {
        super(k);
        value = v;
    }
}
```

可见ThreadLocalMap中基本存储单元为Entry，Entry是一个键值对，key为一个threadlocal，而值由线程设置，ThreadLocalMap中有一个Entry数组名为table用于存放这些Entry。自此，拿到了Entry，若之前已经设置过数据，则直接返回value，否则会执行初始化操作：

```
private T setInitialValue() {

	//initialValue()返回null
    T value = initialValue();
    Thread t = Thread.currentThread();
    
    //首先获得当前线程对应的ThreadLocalMap
    ThreadLocalMap map = getMap(t);
    //若map已经初始化，则设置初始值null
    if (map != null) {
        map.set(this, value);
    } else {
    	//map没有初始化，则创建map并设置初始值null
        createMap(t, value);
    }
    if (this instanceof TerminatingThreadLocal) {
        TerminatingThreadLocal.register((TerminatingThreadLocal<?>) this);
    }
    return value;
}
```

再来看createMap():

```
void createMap(Thread t, T firstValue) {
    t.threadLocals = new ThreadLocalMap(this, firstValue);
}
```

逻辑很简单，创建一个ThreadLocalMap，并初始化值。

总结：get()方法首先会去获取当前线程的ThreadLocalMap，然后再去获取Map中存放的threadlocal对应的值。这一过程中可能会涉及到一些初始化操作。

**ThreadLocal.set():**

再来看set():

```
public void set(T value) {
	//获取当前线程对应的ThreadLocalMap
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null) {
    	//map不为null，则设置threadlocal,value键值对
        map.set(this, value);
    } else {
        createMap(t, value);
    }
}
```

与前述操作基本一致。

ThreadLocal使用：

经过上述分析，其实可以看出一个threadlocal变量可以为不同的线程维护本地变量。

```
public class test {
    public static void main(String[] args) {

        ThreadLocal<String> localStr = new ThreadLocal<>();

        Thread thread1 = new Thread(new Runnable() {
            @Override
            public void run() {
                localStr.set("thread1");
                loaclInt.set(1);
                System.out.println("线程1的localstr: " + localStr.get());
            }
        });
        Thread thread2 = new Thread(new Runnable() {
            @Override
            public void run() {
                localStr.set("thread2");
                loaclInt.set(2);
                System.out.println("线程2的localstr: " + localStr.get());
            }
        });
        thread1.start();
        thread2.start();
```

测试代码中使用同一个localStr为两个线程都维护以他们的本地变量，测试结果如下：

线程1的localstr: thread1
线程2的localstr: thread2
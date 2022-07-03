### 读写锁

ReadWriteLock接口只定义了两个方法

![image-20220625093101641](C:\Users\zhanyu\AppData\Roaming\Typora\typora-user-images\image-20220625093101641.png)

来看他的实现类ReentrantReadWriteLock

ReentrantReadWriteLock使用state的高16位表示读状态，低16位表示写状态。

ReentrantReadWriteLock中包含一个读锁和一个写锁，允许多个线程同时读，但只允许一个线程写，读写锁均支持可重入，因此ReentrantReadWriteLock中的读锁是共享锁，写锁是排他锁，在源码中的体现如下。

**读锁同步状态获取：**

```
public int getReadHoldCount() {
//sync是ReentrantReadWriteLock内部聚合的同步器，支持公平和非公平获取锁
        return sync.getReadHoldCount();
    }
```

下面是sync的getReadHoldCount方法，用于获取当前线程的重入数，由于读锁是共享锁，所以不能简单地返回读锁的总体重入数，因此可能会有多个线程获取了读锁。方法中有四个分支条件：

首先判断ReadLockCount是否为0，是则直接返回；

firstReaderHoldCount保存的是第一个获取读锁的线程的重入数，cachedHoldCounter中保存最后一个成功获取读锁的线程的重入数，若当前线程为第一或最后一个获取读锁的线程，则直接返回所记录的重入数，否则就要进入最后一个分支条件；

readHolds是一个ThreadLocalHoldCounter，它继承了ThreadLocal，这就需要遍历threadMap来获取当前线程的重入数。

```
final int getReadHoldCount() {
            if (getReadLockCount() == 0)
                return 0;

            Thread current = Thread.currentThread();
            if (firstReader == current)
                return firstReaderHoldCount;

            HoldCounter rh = cachedHoldCounter;
            if (rh != null && rh.tid == getThreadId(current))
                return rh.count;

            int count = readHolds.get().count;
            if (count == 0) readHolds.remove();
            return count;
        }
```

再来看getReadLockCount函数，内部调用sharedCount。

```
final int getReadLockCount() {
            return sharedCount(getState());
        }
```

sharedCount接受state值，并对其进行无符号移位操作，这里的SHARED_SHIFT是一个常量值16，所以这里返回的是state的高16位，也即读状态。

```
static int sharedCount(int c)    { return c >>> SHARED_SHIFT; }
```

```
static final int SHARED_SHIFT   = 16;
```

**写锁同步状态获取：**

由于写锁是排他锁，所以获取重入数时只需判断持有锁的线程是否为本线程，若是就可以调用exclusiveCount直接返回重入数。若不是则返回0。

```
public int getWriteHoldCount() {
    return sync.getWriteHoldCount();
}
```

```
final int getWriteHoldCount() {
    return isHeldExclusively() ? exclusiveCount(getState()) : 0;
}
```

这里与之前的读锁类似，只是取低16位返回。

```
static int exclusiveCount(int c) { return c & EXCLUSIVE_MASK; }
```

```
static final int EXCLUSIVE_MASK = (1 << SHARED_SHIFT) - 1;
```

**非阻塞的获取读锁**

readLock的tryLock方法会以非阻塞方式获取读锁，在其中调用的是sync.tryReadLock。

```
public boolean tryLock() {
    return sync.tryReadLock();
}
```

sync.tryReadLock以自旋方式获取读锁。

首先判断写状态是否为0，如果不为0说明有写线程持有锁，但这时并不是一定无法获取读锁，还需要查看持有写锁的线程是否为当前线程，如果是则仍然可以获取读锁，也即一个线程可以同时拥有读锁和写锁。只有当非当前的线程持有写锁时才会返回false表示无法获取锁。

通过了判断条件后即可获取读锁，可以看到，读锁的获取不会setExclusiveOwnerThread，只会更改state值并维护firstReader、firstReaderHoldCount和cachedHoldCounter，主要作用是加快读锁重入数的获取

```
final boolean tryReadLock() {
    Thread current = Thread.currentThread();
    for (;;) {
        int c = getState();
        if (exclusiveCount(c) != 0 &&
            getExclusiveOwnerThread() != current)
            return false;
        int r = sharedCount(c);
        if (r == MAX_COUNT)
            throw new Error("Maximum lock count exceeded");
        if (compareAndSetState(c, c + SHARED_UNIT)) {
            if (r == 0) {
                firstReader = current;
                firstReaderHoldCount = 1;
            } else if (firstReader == current) {
                firstReaderHoldCount++;
            } else {
                HoldCounter rh = cachedHoldCounter;
                if (rh == null || rh.tid != getThreadId(current))
                    cachedHoldCounter = rh = readHolds.get();
                else if (rh.count == 0)
                    readHolds.set(rh);
                rh.count++;
            }
            return true;
        }
    }
}
```

**阻塞的获取读锁**

readLock的lock方法会以阻塞方式获取读锁，在其中调用的是sync.acquireShared。

```
public void lock() {
    sync.acquireShared(1);
}
```

acquireShared中首先调用tryAcquireShared方法获取读锁，若是失败，则调用doAcquireShared方法，以自旋的方式获取读锁。

```
public final void acquireShared(int arg) {
    if (tryAcquireShared(arg) < 0)
        doAcquireShared(arg);
}
```

先来看下tryAcquireShared方法：

首先判断是否有写线程持有写锁，且比较持有写锁的线程是否为当前线程，如果是则可以继续获取读锁如果不是则获取失败，返回-1；

如果通过判断条件则修改state，并维护firstReader、firstReaderHoldCount和cachedHoldCounter。

如果设置不成功，则会进入fullTryAcquireShared，以自旋方式获取读锁，内部逻辑与tryAcquireShared基本是相同的。

```
protected final int tryAcquireShared(int unused) {
 
    Thread current = Thread.currentThread();
    int c = getState();
   
    if (exclusiveCount(c) != 0 &&
        getExclusiveOwnerThread() != current)
        return -1;
    int r = sharedCount(c);
    if (!readerShouldBlock() &&
        r < MAX_COUNT &&
        compareAndSetState(c, c + SHARED_UNIT)) {
        if (r == 0) {
            firstReader = current;
            firstReaderHoldCount = 1;
        } else if (firstReader == current) {
            firstReaderHoldCount++;
        } else {
            HoldCounter rh = cachedHoldCounter;
            if (rh == null || rh.tid != getThreadId(current))
                cachedHoldCounter = rh = readHolds.get();
            else if (rh.count == 0)
                readHolds.set(rh);
            rh.count++;
        }
        return 1;
    }
    return fullTryAcquireShared(current);
}
```

**非阻塞的获取写锁**

tryWriteLock中，若以有获取了读锁的线程或者已有其他获取写锁的线程，都会获取失败。也就意味着获取了读锁的线程不能再获取写锁。

通过判断条件后便使用CAS方式设置state并setExclusiveOwnerThread

```
public boolean tryLock( ) {
    return sync.tryWriteLock();
}
```

```
final boolean tryWriteLock() {
    Thread current = Thread.currentThread();
    int c = getState();
    if (c != 0) {
        int w = exclusiveCount(c);
        if (w == 0 || current != getExclusiveOwnerThread())
            return false;
        if (w == MAX_COUNT)
            throw new Error("Maximum lock count exceeded");
    }
    if (!compareAndSetState(c, c + 1))
        return false;
    setExclusiveOwnerThread(current);
    return true;
}
```

**阻塞的获取写锁**

```
public void lock() {
    sync.acquire(1);
}
```

acquire的处理过程其实就是调用AQS框架中的tryAcquire,失败后以addWaiter并加入同步队列，在acquireQueued中自旋的获取同步状态。

```
public final void acquire(int arg) {
    if (!tryAcquire(arg) &&
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
}
```
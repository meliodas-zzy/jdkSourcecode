### 可重入锁

表示支持一个线程对某个资源进行重复加锁

ReentrantLock也是通过实现Lock接口并在内部聚合一个同步器来实现的，且支持公平和非公平性获取锁。所谓公平性获取锁指在同步队列中靠前的等待线程会先获得锁。

```
public class ReentrantLock implements Lock, java.io.Serializable {
    private static final long serialVersionUID = 7373984872572414699L;
    
    private final Sync sync;
    
    abstract static class Sync extends AbstractQueuedSynchronizer {
        final boolean nonfairTryAcquire(int acquires) {...}
        protected final boolean tryRelease(int releases) {...} 
        ...
    }
    
    static final class NonfairSync extends Sync {...}
    static final class FairSync extends Sync {...}
}

```

#### 非公平性获取锁

每个线程都会以CAS方式更改state值以获取锁，且函数会接受一个int值表示获取锁的数量。

可以看出，如果state为0，表示没有线程持有锁，直接进行获取锁操作，若state不为0，则检查持有锁的线程是否为本线程，若是，则可以重复加锁，这就实现了可重入的功能。

```
final boolean nonfairTryAcquire(int acquires) {
    final Thread current = Thread.currentThread();
    int c = getState();
    if (c == 0) {
        if (compareAndSetState(0, acquires)) {
            setExclusiveOwnerThread(current);
            return true;
        }
    }
    else if (current == getExclusiveOwnerThread()) {
        int nextc = c + acquires;
        if (nextc < 0) // overflow
            throw new Error("Maximum lock count exceeded");
        setState(nextc);
        return true;
    }
    return false;
}
```

可重入的获取锁决定了state值最大是可以超过1的，因此需要在release时进行判断,只有state值减到0时才会释放锁，若state不为0，那么tryRelease永远会返回false

```
protected final boolean tryRelease(int releases) {
            int c = getState() - releases;
            if (Thread.currentThread() != getExclusiveOwnerThread())
                throw new IllegalMonitorStateException();
            boolean free = false;
            if (c == 0) {
                free = true;
                setExclusiveOwnerThread(null);
            }
            setState(c);
            return free;
        }
```

#### 公平性获取锁

相较于非公平性获取锁，加入了一个判断条件!hasQueuedPredecessors()，表示只有当前线程在同步队列中没有前驱节点时才能获取锁，这就保证了公平性

```
protected final boolean tryAcquire(int acquires) {
            final Thread current = Thread.currentThread();
            int c = getState();
            if (c == 0) {
                if (!hasQueuedPredecessors() &&
                    compareAndSetState(0, acquires)) {
                    setExclusiveOwnerThread(current);
                    return true;
                }
            }
            else if (current == getExclusiveOwnerThread()) {
                int nextc = c + acquires;
                if (nextc < 0)
                    throw new Error("Maximum lock count exceeded");
                setState(nextc);
                return true;
            }
            return false;
        }
```


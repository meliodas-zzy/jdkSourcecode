### **AQS框架**

锁的实现者只需要重写AQS中提供的以下方法即可简便的实现一个锁

| 方法                                | 功能         |
| ----------------------------------- | ------------ |
| tryAcquire(int arg) : boolean       | 独占式获取锁 |
| tryRelease(int arg) : boolean       | 独占式释放锁 |
| tryAcquireShared(int arg) : int     | 共享式获取锁 |
| tryReleaseShared(int arg) : boolean | 共享式释放锁 |

重写这些方法时需要调用以下方法来访问和修改同步状态

| 方法                                                 | 功能                                      |
| ---------------------------------------------------- | ----------------------------------------- |
| getState() : int                                     | state即为同步状态，此方法用于获取同步状态 |
| setState(int) : void                                 | 设置同步状态                              |
| compareAndSetState(int expect, int update) : boolean | CAS方式设置同步状态                       |

AQS框架提供了一些模板方法为使用者屏蔽了底层同步状态管理，线程在队列中的排队，等待和唤醒等底层操作

| 方法                                                        | 功能                                                         |
| ----------------------------------------------------------- | ------------------------------------------------------------ |
| acquire() : void                                            | 独占式获取锁，该方法将会调用重写的tryAcquire()方法，如果成功获取锁则返回，失败则线程进入同步等待队列 |
| acquireInterruptibly(int arg) : void                        | 独占式获取锁，但是该方法可以响应中断                         |
| tryAcquireNanos(int arg, long nanosTimeout) : boolean       | 独占式超时获取锁且能够响应中断，若在指定时间内获取锁失败则返回false |
| acquireShared(int arg)                                      | 共享式获取锁，与独占式的区别为共享式允许同一时间可以有多个线程获取到锁 |
| acquireSharedInterruptibly(int arg) : void                  | 共享式获取锁，且响应中断                                     |
| tryAcquireSharedNanos(int arg, long nanosTimeout) : boolean | 共享式超时获取锁，且响应中断                                 |
| release(int arg) : boolean                                  | 独占式释放锁                                                 |
| acquireShared(int arg) : void                               | 共享式获取锁                                                 |

可重写的方法中按照锁实现者的意愿来实现不同功能的锁。

通过源码查看AQS如何为使用者屏蔽底层细节

#### 独占式获取锁：

```
public final void acquire(int arg) {
    if (!tryAcquire(arg) &&
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))//Node.EXCLUSIVE表示处于独占状态
        selfInterrupt();
}
```

acquire方法首先调用tryAcquire方法获取锁，如果失败则调用addWaiter方法构建一个Node节点并调用acquireQueued方法将其加入同步队列队尾，acquireQueued不响应中断，直到返回到acquire才会对中断进行处理。Node为同步节点，用于将线程放置到同步队列中。Node主要属性如下

##### int waitStatus:

CANCELLED：值为1，表示线程等待超时或者被中断，需要从同步队列中取消等待

SIGNAL ：值为-1。后继节点处于等待状态，当前节点（为-1）被取消或者中断时会通知后继节点，使后继节点的线程得以运行

CONDITION ：值为-2.当前节点处于等待队列，节点线程等待在Condition上，当其他线程对condition执行signall方法时，等待队列转移到同步队列，加入到对同步状态的获取

PROPAGATE：值为-3，与共享模式相关，在共享模式中，该状态标识结点的线程处于可运行状态

0状态：值为0，代表初始化状态

##### Node prev 

前驱节点

##### Node next

后继节点

##### Node nextWaiter:

Node节点既可以在同步队列中，也可以在等待队列中，在同步队列中时，nextWaiter可能有两个值：EXCLUSIVE表示处于独占模式，SHARED表示处于共享模式。在等待队列中时nextWaiter保存后继节点

##### Thread thread

节点中的线程

addWaiter中的实现逻辑为

```
private Node addWaiter(Node mode) {
    Node node = new Node(Thread.currentThread(), mode);
    // Try the fast path of enq; backup to full enq on failure
    Node pred = tail;
    //尝试快速的将节点加入到队列队尾
    if (pred != null) {
        node.prev = pred;
        if (compareAndSetTail(pred, node)) {
            pred.next = node;
            return node;
        }
    }
    //快速加入失败后，以自旋方式不断尝试
    enq(node);
    return node;
}
```

再来看enq函数源码

```
private Node enq(final Node node) {
    for (;;) {
        Node t = tail;//获取同步队列尾节点
        if (t == null) { // 队列为空时，必须先初始化
            if (compareAndSetHead(new Node()))
                tail = head;
        } else {
            node.prev = t;
            //自旋的使用CAS方式将当前节点设置为尾节点
            if (compareAndSetTail(t, node)) {
                t.next = node;
                return t;
            }
        }
    }
}
```

当将节点以独占式加入到同步队列队尾后，调用acquireQueued方法，使节点进入自旋状态，当条件满足时，就获取同步状态。条件满足指的是当前节点的前驱节点为首节点。当线程获取同步状态失败后，会调用shouldParkAfterFailedAcquire及parkAndCheckInterrupt，对节点的状态进行处理及设置线程为阻塞状态。

```
final boolean acquireQueued(final Node node, int arg) {
    boolean failed = true;
    try {
        boolean interrupted = false;
        for (;;) {
            final Node p = node.predecessor();
            //条件满足，获取同步状态
            if (p == head && tryAcquire(arg)) {
                setHead(node);
                p.next = null; // help GC
                failed = false;
                return interrupted;
            }
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                interrupted = true;
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
```

下面来看shouldParkAfterFailedAcquire及parkAndCheckInterrupt内部逻辑

```
private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
        int ws = pred.waitStatus;
        /* 有一个需要遵循的原则：只有前置节点状态为SIGNAL时，
         * 即代表前置节点在被取消或者中断时会通知当前节点获取同步状态，
         * 只有这种情况下当前节点才能放心的挂起
         */
        if (ws == Node.SIGNAL)
            /*
             * 前置节点已经是SIGNAL，前置节点会在之后通知当前节点获取同步状态
             * 因此当前节点可以放心的挂起
             */
            return true;
        if (ws > 0) {
            /*
             *前置节点处于CANCELLED状态，此时需要删除所有的CANCELLED状态节点
             */
            do {
                node.prev = pred = pred.prev;
            } while (pred.waitStatus > 0);
            pred.next = node;
        } else {
            /*
             * 前置节点状态为 0 or PROPAGATE，此时先使用CAS方式设置SIGNAL，
             * 但并不立即挂起当前线程，而是先去尝试获取同步状态
             */
            compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
        }
        return false;
    }
```

设置好状态后就可以执行挂起线程的操作，调用LockSupport的park方法挂起线程

```
private final boolean parkAndCheckInterrupt() {
        LockSupport.park(this);
        return Thread.interrupted();
    }
```

#### 独占式释放同步状态

调用自定义同步器组件实现的tryRelease方法成功后，检测头节点状态，unparkSuccessor用于唤醒后继节点

```
public final boolean release(int arg) {
    if (tryRelease(arg)) {
        Node h = head;
        if (h != null && h.waitStatus != 0)
            unparkSuccessor(h);
        return true;
    }
    return false;
}
```

```
private void unparkSuccessor(Node node) {
        /*
         * 将当前节点状态以CAS方式设置为0 
         */
        int ws = node.waitStatus;
        if (ws < 0)
            compareAndSetWaitStatus(node, ws, 0);

        /*
         * 首先判断后继节点是否为可以唤醒的有效节点，若不是，则
         * 从队尾向前遍历，找出最靠前的有效节点，调用LockSupport.unpark方法执行唤醒操作
         */
        Node s = node.next;
        if (s == null || s.waitStatus > 0) {
            s = null;
            for (Node t = tail; t != null && t != node; t = t.prev)
                if (t.waitStatus <= 0)
                    s = t;
        }
        if (s != null)
            LockSupport.unpark(s.thread);
    }
```

#### 共享式获取锁

自定义同步器组件实现tryAcquireShared方法，将返回一个int类型值，为0表示获取同步状态成功，但已经没有剩余资源可以获取，大于0表示获取同步状态成功，仍有剩余资源可以获取，为负数表示获取同步状态失败，获取同步状态失败时将执行doAcquireShared方法，以自旋方式获取同步状态

```
public final void acquireShared(int arg) {
        if (tryAcquireShared(arg) < 0)
            doAcquireShared(arg);
    }
```

```
private void doAcquireShared(int arg) {
	//和独占式类似，先构建Node节点，加入到同步队列中
    final Node node = addWaiter(Node.SHARED);
    boolean failed = true;
    try {
        boolean interrupted = false;
        for (;;) {
            final Node p = node.predecessor();
            if (p == head) {
                int r = tryAcquireShared(arg);
                //获取同步状态成功
                if (r >= 0) {
                    setHeadAndPropagate(node, r);
                    p.next = null; // help GC
                    if (interrupted)
                        selfInterrupt();
                    failed = false;
                    return;
                }
            }
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                interrupted = true;
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
```

```
private void setHeadAndPropagate(Node node, int propagate) {
        Node h = head; // Record old head for check below
        setHead(node);
        /*
         * 这里与独占式不同，若仍有资源可以获取则执行唤醒操作
         */
        if (propagate > 0 || h == null || h.waitStatus < 0 ||
            (h = head) == null || h.waitStatus < 0) {
            Node s = node.next;
            if (s == null || s.isShared())
                doReleaseShared();
        }
    }
```


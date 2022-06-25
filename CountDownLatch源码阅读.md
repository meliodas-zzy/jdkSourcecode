**CountDownLatch**

CountDownLatch允许一个或多个线程等待其他线程完成操作后再继续执行

CountDownLatch在内部聚合一个AQS，实现了自定义的tryAcquireShared和tryReleaseShared方法，其构造方法接受一个int值，要等待多少个节点完成任务，就传入相应的值。在使用时只需在每个线程中调用countDown方法，就会将count-1，直至count减为0，就会通知处于等待状态的线程，让其继续工作。

```
public CountDownLatch(int count) {
    if (count < 0) throw new IllegalArgumentException("count < 0");
    this.sync = new Sync(count);
}
```

**await()**

调用await()的线程将等待其他线程工作完成后再继续工作

```
public void await() throws InterruptedException {
    sync.acquireSharedInterruptibly(1);
}
```

在其内部调用的是AQS的acquireSharedInterruptibly方法，以共享方式获取同步状态

```
public final void acquireSharedInterruptibly(int arg)
        throws InterruptedException {
    if (Thread.interrupted())
        throw new InterruptedException();
    if (tryAcquireShared(arg) < 0)
        doAcquireSharedInterruptibly(arg);
}
```

而acquireSharedInterruptibly内部调用的是CountDownLatch实现的tryAcquireShared方法，很明显的是只是简单的判断一下count是否为0，而没有去更改state值。其实只有在执行countdown操作时，才会去修改count值。当count不为0，即仍有线程未完成工作时，执行doAcquireSharedInterruptibly

```
protected int tryAcquireShared(int acquires) {
    return (getState() == 0) ? 1 : -1;
}
```

在doAcquireSharedInterruptibly中，首先构建节点并将线程加入到同步队列，之后以自旋的方式获取同步状态。CountDownLatch对tryAcquireShared的实现机制决定了只有将count的值countdown为0时，线程才能够获取同步状态否则挂起。

```
private void doAcquireSharedInterruptibly(int arg)
    throws InterruptedException {
    //构建节点，加入到同步队列
    final Node node = addWaiter(Node.SHARED);
    boolean failed = true;
    try {
        for (;;) {
            final Node p = node.predecessor();
            if (p == head) {
            	//前驱节点时头节点，尝试去获取同步状态
                int r = tryAcquireShared(arg);
                //CountDownLatch的tryAcquireShared只有count为0时才返回1，其余都返回-1
                if (r >= 0) {
                    setHeadAndPropagate(node, r);
                    p.next = null; // help GC
                    failed = false;
                    return;
                }
            }
            //获取同步状态失败，挂起线程
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                throw new InterruptedException();
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
```

**countDown()**

```
public void countDown() {
    sync.releaseShared(1);
}
```

releaseShared调用CountDownLatch实现的tryReleaseShared方法

```
public final boolean releaseShared(int arg) {
    if (tryReleaseShared(arg)) {
        doReleaseShared();
        return true;
    }
    return false;
}
```

tryReleaseShared以自旋的CAS方式将count减一，可以看到count不为0时总是返回false，因此只有将count减为0时才会执行doReleaseShared方法

```
protected boolean tryReleaseShared(int releases) {
    // Decrement count; signal when transition to zero
    for (;;) {
        int c = getState();
        //若count原本就是0，则会返回false
        if (c == 0)
            return false;
        int nextc = c-1;
        //只有减1后为0时，才会返回true
        if (compareAndSetState(c, nextc))
            return nextc == 0;
    }
}
```

在doReleaseShared方法中，以自旋方式唤醒等待线程。

```
private void doReleaseShared() {
    
    for (;;) {
        Node h = head;
        if (h != null && h != tail) {
            int ws = h.waitStatus;
            if (ws == Node.SIGNAL) {
                if (!compareAndSetWaitStatus(h, Node.SIGNAL, 0))
                    continue;            // loop to recheck cases
                unparkSuccessor(h);
            }
            else if (ws == 0 &&
                     !compareAndSetWaitStatus(h, 0, Node.PROPAGATE))
                continue;                // loop on failed CAS
        }
        if (h == head)                   // loop if head changed
            break;
    }
}
```

**总结：**

通过实现tryAcquireShared和tryReleaseShared方法，CountDownLatch实现了当线程调用await方法时不改变count值，只有在执行countDown方法时，才会将count值减一，直至count为0时，就会将等待线程唤醒。
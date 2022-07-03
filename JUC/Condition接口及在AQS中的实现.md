**Condition接口**

定义了类似Object对象的监视器方法的await(),signal()等，与Lock对象配合可以实现等待/通知模型，Condition对象是依赖于Lock对象的。AbstractQueuedSynchronizer中的ConditionObject就实现了Condition接口。

一般都会将Condition对象作为一个成员变量使用，当调用Condition.await()方法后，当前线程会释放锁并等待，当调用Condition.signal()方法后将通知等待的线程，等待线程会从await()方法返回。

ConditionObject内主要成员及方法有

```
public class ConditionObject implements Condition, java.io.Serializable {
	private transient Node firstWaiter;
	private transient Node lastWaiter;
	private void doSignal(Node first) {...}
	private void doSignalAll(Node first) {...}
	public final void signal() {...}
	public final void signalAll() {...}
	public final void await() throws InterruptedException {...}
	public final boolean await(long time, TimeUnit unit)
                throws InterruptedException {...}
    ...
}
```

可以看到复用了AQS的Node，且维护了一个等待队列，等待队列中的节点获取后继节点需要使用nextWaiter而不是next，这一点与同步队列中的节点不同。

**await()**

await方法执行流程为：

调用addConditionWaiter将节点加入到等待队列中

调用fullyRelease释放锁

对于不在同步队列中的节点，可以放心地挂起，进入一个循环等待的流程，这一步很关键

acquireQueued获取同步状态

清理处于CANCELLED状态的节点

处理中断

```
public final void await() throws InterruptedException {
    if (Thread.interrupted())
        throw new InterruptedException();
    Node node = addConditionWaiter();
    int savedState = fullyRelease(node);
    int interruptMode = 0;
    while (!isOnSyncQueue(node)) {
        LockSupport.park(this);
        if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
            break;
    }
    if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
        interruptMode = REINTERRUPT;
    if (node.nextWaiter != null) // clean up if cancelled
        unlinkCancelledWaiters();
    if (interruptMode != 0)
        reportInterruptAfterWait(interruptMode);
}
```

首先看一下addConditionWaiter

```
private Node addConditionWaiter() {
    Node t = lastWaiter;
    // 清理CANCELLED状态的节点
    if (t != null && t.waitStatus != Node.CONDITION) {
        unlinkCancelledWaiters();
        t = lastWaiter;
    }
    //构造节点，节点状态为CONDITION
    Node node = new Node(Thread.currentThread(), Node.CONDITION);
    if (t == null)
        firstWaiter = node;
    else
        t.nextWaiter = node;
    lastWaiter = node;
    return node;
}
```

加入到等待队列中后，就要释放锁了

```
final int fullyRelease(Node node) {
    boolean failed = true;
    try {
        int savedState = getState();
        //调用release方法释放锁，之前已经分析过
        if (release(savedState)) {
            failed = false;
            return savedState;
        } else {
            throw new IllegalMonitorStateException();
        }
    } finally {
        if (failed)
            node.waitStatus = Node.CANCELLED;
    }
}
```

接下来的循环跳出条件有两种，一种是isOnSyncQueue(node)为true，当其他线程执行signal操作后会将等待线程移动到同步队列；第二种是当前线程被中断，执行break。

```
while (!isOnSyncQueue(node)) {
    LockSupport.park(this);
    if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
        break;
}
```

checkInterruptWhileWaiting顾名思义就是检测线程在等待过程中是否被中断过，如果没有，则返回0，如果被中断，又分为两种情况，接下来分析transferAfterCancelledWait得出结论。

```
private int checkInterruptWhileWaiting(Node node) {
    return Thread.interrupted() ?
        (transferAfterCancelledWait(node) ? THROW_IE : REINTERRUPT) :
        0;
}
```

线程被唤醒可能是因为被中断，也可能是因为其他线程执行了signal操作，在执行signal操作时，会设置线程状态值为0，根据这一点就可以判断线程被唤醒的原因。

```
final boolean transferAfterCancelledWait(Node node) {
	/*
	*如果这一步成功，说明线程之前的状态是CONDITION，那么可以使用CAS操作将状态设置为0
	*然后调用enq方法将线程加入到同步队列
	*/
    if (compareAndSetWaitStatus(node, Node.CONDITION, 0)) {
        enq(node);
        return true;
    }
    /*
     *如果执行到这里，说明有线程执行了signal操作，那么就不需要由当前线程执行enq操作，
     *而是将这一操作交给执行signal操作的线程（在signal的流程中就可以看到），
     *当前线程秩序执行yield让出CPU即可
     */
    while (!isOnSyncQueue(node))
        Thread.yield();
    return false;
}
```

transferAfterCancelledWait返回true说明有中断产生，所以checkInterruptWhileWaiting返回THROW_IE（-1），之后就根据interruptMode的来进行相应的处理，并且需要清理处于CANCELLED状态的节点

```
if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
    interruptMode = REINTERRUPT;
if (node.nextWaiter != null) // clean up if cancelled
    unlinkCancelledWaiters();
if (interruptMode != 0)
    reportInterruptAfterWait(interruptMode);
```

**signal()**

相对于await方法，signal要简单一些。

首先进行判断，只有锁的持有者才能执行唤醒操作

doSignal执行真正的唤醒操作

```
public final void signal() {
    if (!isHeldExclusively())
        throw new IllegalMonitorStateException();
    Node first = firstWaiter;
    if (first != null)
        doSignal(first);
}
```

doSignal函数内是一个循环，找出第一个需要唤醒的节点

```
private void doSignal(Node first) {
    do {
        if ( (firstWaiter = first.nextWaiter) == null)
            lastWaiter = null;
        first.nextWaiter = null;
    } while (!transferForSignal(first) &&
             (first = firstWaiter) != null);
}
```

在这里修改节点状态，并将节点加入到同步队列中

```
final boolean transferForSignal(Node node) {
    /*
     * If cannot change waitStatus, the node has been cancelled.
     */
    if (!compareAndSetWaitStatus(node, Node.CONDITION, 0))
        return false;

    /*
     * 加入同步队列
     */
    Node p = enq(node);
    int ws = p.waitStatus;
 	//判断此时阻塞队列是否有前继节点等待，有就Park线程，等待唤醒
    if (ws > 0 || !compareAndSetWaitStatus(p, ws, Node.SIGNAL))
        LockSupport.unpark(node.thread);
    return true;
}
```

**signalAll()**

基本上与signal流程一致，只是在调用doSignal时改为调用doSignalAll,通过循环将所有的等待节点唤醒

```
private void doSignalAll(Node first) {
    lastWaiter = firstWaiter = null;
    do {
        Node next = first.nextWaiter;
        first.nextWaiter = null;
        transferForSignal(first);
        first = next;
    } while (first != null);
}
```
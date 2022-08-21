### Runnable

由将要由thread执行的某个类实现，只提供一个run()方法，无返回值，可以在Thread和线程池中使用且不会抛出异常。

### Callable

由将要由thread执行的某个类实现，只提供一个run()方法，有返回值，只能在线程池中使用，且会抛出异常。

Executors中提供了将Runnable对象转换为Callable的方法:

```
public static <T> Callable<T> callable(Runnable task, T result) {
    if (task == null)
        throw new NullPointerException();
    return new RunnableAdapter<T>(task, result);
}
```

### Future

代表异步计算的计算结果。提供了一系列方法检查计算是否已经完成，记忆获取计算结果等。

**get()**:用于获取计算结果，若计算尚未完成，则会等待直到计算完成。

**isDone()**:当任务结束时，返回true。引起任务结束的原因有正常结束，异常结束。

**cancel()**:尝试去取消任务，当任务已经完成或者有已经被取消或者因为某些原因无法被取消时会失败。当一个尚未开始的任务执行cancel()后，这个任务再也不会执行。值得注意的是，当cancel()返回true后，isDone()方法和isCancelled()均会返回true。

**isCancelled():**当任务结束前被取消了，则返回true。




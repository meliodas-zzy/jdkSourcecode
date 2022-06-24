**Lock接口**

| 方法 ：返回值                     | 功能                                                         |
| --------------------------------- | ------------------------------------------------------------ |
| lock() : void                     | 获取锁后返回                                                 |
| lockInterruptibly() ： void       | 该方法在获取锁时能够响应中断                                 |
| tryLock() ： boolean              | 非阻塞的获取锁，成功获取锁时返回true，否则返回false          |
| tryLock(long time, TimeUnit unit) | 超时的获取锁                                                 |
| unlock() :  void                  | 释放锁                                                       |
| newCondition() ：Condition        | 获取一个和Lock绑定的通知组件，调用Condition的wait方法后将释放锁 |


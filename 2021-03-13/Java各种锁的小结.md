### Java各种锁的小结

#### 1.synchronized

在 JDK 1.6 之前，synchronized 是重量级锁，效率低下。

从 JDK 1.6 开始，synchronized 做了很多优化，如偏向锁、轻量级锁、自旋锁、适应性自旋锁、锁消除、锁粗化等技术来减少锁操作的开销。

synchronized 同步锁一共包含四种状态：无锁、偏向锁、轻量级锁、重量级锁，它会随着竞争情况逐渐升级。synchronized 同步锁可以升级但是不可以降级，目的是为了提高获取锁和释放锁的效率。

**synchronized的底层原理**

- synchronized修饰的代码块

通过反编译.class文件，通过查看字节码可以得到：在代码块中使用的是 monitorenter 和 monitorexit 指令，其中 monitorenter 指令指向同步代码块的开始位置，monitorexit 指令指明同步代码块的结束位置。

- synchronized修饰的方法

同样查看字节码可以得到：在同步方法中会包含 ACC_SYNCHRONIZED 标记符。该标记符指明了该方法是一个同步方法，从而执行相应的同步调用。

#### 2. 对象锁，类锁，私有锁

- 对象锁：使用 synchronized 修饰非静态的方法以及 synchronized(this) 同步代码块使用的锁是对象锁。
- 类锁：使用 synchronized 修饰静态的方法以及 synchronized(class) 同步代码块使用的锁是类锁。
- 私有锁：在类内部声明一个私有属性如private Object lock，在需要加锁的同步块使用 synchronized(lock)。

他们的特性：

- 对象锁具有可重入性。
- 当一个线程获得了某个对象的对象锁，则该线程仍然可以调用其他任何需要该对象锁的 synchronized 方法或 synchronized(this) 同步代码块。
- 当一个线程访问某个对象的一个 synchronized(this) 同步代码块时，其他线程对该对象中所有其它 synchronized(this) 同步代码块的访问将被阻塞，因为访问的是同一个对象锁。
- 每个类只有一个类锁，但是类可以实例化成对象，因此每一个对象对应一个对象锁。
- 类锁和对象锁不会产生竞争。
- 私有锁和对象锁也不会产生竞争。
- 使用私有锁可以减小锁的细粒度，减少由锁产生的开销。

由私有锁实现的等待/通知机制：

```java
Object lock = new Object();// 由等待方线程实现
synchronized (lock) {
    while (条件不满足) {
       lock.wait();
   }                         
}// 由通知方线程实现
synchronized (lock) {
   条件发生改变
   lock.notify();                    
}
```

#### 3. ReentrantLock

ReentrantLock 是一个独占/排他锁。相对于 synchronized，它更加灵活。但是需要自己写出加锁和解锁的过程。它的灵活性在于它拥有很多特性。

> ReentrantLock 需要显示地进行释放锁。特别是在程序异常时，synchronized 会自动释放锁，而 ReentrantLock 并不会自动释放锁，所以必须在 finally 中进行释放锁。

它的特性：

- 公平性：支持公平锁和非公平锁。默认使用了非公平锁。
- 可重入
- 可中断：相对于 synchronized，它是可中断的锁，能够对中断作出响应。
- 超时机制：超时后不能获得锁，因此不会造成死锁。

ReentrantLock 是很多类的基础，例如 ConcurrentHashMap 内部使用的 Segment 就是继承 ReentrantLock，CopyOnWriteArrayList 也使用了 ReentrantLock。

#### 4.ReentantReadWrite

它拥有读锁(ReadLock)和写锁(WriteLock)，读锁是一个共享锁，写锁是一个排他锁。

它的特性：

- 公平性：支持公平锁和非公平锁。默认使用了非公平锁。
- 可重入：读线程在获取读锁之后能够再次获取读锁。写线程在获取写锁之后能够再次获取写锁，同时也可以获取读锁（锁降级）。
- 锁降级：先获取写锁，再获取读锁，然后再释放写锁的过程。锁降级是为了保证数据的可见性。

https://www.imooc.com/article/274818

#### 5.CAS

上面提到的 ReentrantLock、ReentrantReadWriteLock 都是基于 AbstractQueuedSynchronizer (AQS)，而 AQS 又是基于 CAS。CAS 的全称是 Compare And Swap（比较与交换），它是一种无锁算法。

synchronized、Lock 都采用了悲观锁的机制，而 CAS 是一种乐观锁的实现。

CAS 的特性：

- 通过调用 JNI 的代码实现
- 非阻塞算法
- 非独占锁

CAS 存在的问题：

- ABA
- 循环时间长开销大
- 只能保证一个共享变量的原子操作

#### 6.Condition

Condition 用于替代传统的 Object 的 wait()、notify() 实现线程间的协作。

> 在 Condition 对象中，与 wait、notify、notifyAll 方法对应的分别是 await、signal 和 signalAll。

Condition 必须要配合 Lock 一起使用，一个 Condition 的实例必须与一个 Lock 绑定。

它的特性：

- 一个 Lock 对象可以创建多个 Condition 实例，所以可以支持多个等待队列。
- Condition 在使用 await、signal 或 signalAll 方法时，必须先获得 Lock 的 lock()
- 支持响应中断
- 支持的定时唤醒功能

#### 7.Semaphore

Semaphore、CountDownLatch、CyclicBarrier 都是并发工具类。

Semaphore 可以指定多个线程同时访问某个资源，而 synchronized 和 ReentrantLock 都是一次只允许一个线程访问某个资源。由于 Semaphore 适用于限制访问某些资源的线程数目，因此可以使用它来做限流。

> Semaphore 并不会实现数据的同步，数据的同步还是需要使用 synchronized、Lock 等实现。

它的特性：

- 基于 AQS 的共享模式
- 公平性：支持公平模式和非公平模式。默认使用了非公平模式。

#### 8.CountDownLatch

CountDownLatch 可以看成是一个倒计数器，它允许一个或多个线程等待其他线程完成操作。因此，CountDownLatch 是共享锁。

CountDownLatch 的 countDown() 方法将计数器减1，await() 方法会阻塞当前线程直到计数器变为0。

#### 9.ConcurrentHashMap的分段锁Segment

#### 10.分布式锁

https://www.cnblogs.com/seesun2012/p/9214653.html
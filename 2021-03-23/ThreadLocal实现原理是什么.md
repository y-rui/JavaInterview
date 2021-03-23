### ThreadLocal 实现原理是什么

#### 1.概述

ThreadLocal是java.lang包中的一个类，用来实现变量的线程封闭性，即只有当前线程可以操作该变量，通过把一个变量存在当前线程的一个Map容器中来实现。当然，这样解释很抽象，对于一些人来说难以理解，这里我先介绍一个《java并发编程实战》中描述的jdbc应用场景，来让你知道ThreadLocal到底有什么用。

#### 2.ThreadLocal的意义

我们知道，当在普通方法中创建一个变量类，若没有特别在方法区域外留有该类的引用，当方法结束后在其它地方不能够再使用这个类。当我们在其它地方还要用到方法中创建的对象时，我们通常会用一个全局变量指向这个对象，这样在整个项目中都能再访问这个对象了。但想想，在多线程情况下，每个线程访问该方法都会创建一个全局对象，在高并发下那我们岂不是要创建成千上万个全局变量来存？若该对象还带有每个线程特有的参数，那就要保证每个线程在之后能调用自己的创建的对象，一般情况下是很难进行管理的。而ThreadLocal就做到了既能让对象在其它地方被创建线程访问，也省去了自己管理全局对象的麻烦。

一般来讲，在单线程应用程序中可能会维持一个全局的数据库连接，并在程序启动时初始化这个连接对象，从而避免在调用每个方法(save,get等)时都要传递一个Connection对象。由于Jdbc连接对象不一定是线程安全的，因此，当多线程应用程序在没有协同的情况下使用全局变量时，就不是线程安全的，比如线程一刚获取全局的Connection，准备进行数据库操作，但线程二却执行了Connection.close()。

这种情况下我们可能会取消Connection这个全局变量，在每次要进行数据库相关操作时直接new一个Connection对象进行连接，而这又会导致一个线程执行多次数据库操作时要new多个connection对象，加大系统的负担。这时就会想能不能有这样一种方法，既不让Connection成为全局变量来保证线程安全，又可以实现全局Connection带来的“一次连接”式的便利？ThreadLocal就是做这件事的。

#### 3.简要应用

下面第一行代码new了一个静态的ThreadLocal<Connection>。以后只要让每个线程在进行jdbc操作前都执行第二行代码，就会把一个创建的Connection对象存放到当前线程的map容器中（ThreadLocalMap，见下文介绍），后面该线程要进行数据库操作时只要执行第三行代码，就能拿到这个引用进行连接，不用每次都new一个新的。

```java
private static ThreadLocal<Connection> connectionHolder = new ThreadLocal<>();
 
connectionHolder.set(DriverManager.getConnection(DB_URL));
 
connectionHolder.get();
```

你可能会问只是一句简单的connectionHolder.get()代码，而且ThreadLocal对象只有一个，那怎么能准确的拿到当前线程中的Connection呢？答案就是：这个connection是存在当前线程（一个Thread）中的，不是ThreadLocal中。表面上调用的ThreadLocal.get()，实际上是在当前线程对象的Map容器中进行设置和查找。不过当前线程的map容器中可能会存多个ThreadLocal的值，所以Map中的key就是ThreadLoca对象，值就是connection，来进行区分。还不懂的话见下面的代码解析，很简单。

#### 4.源代码分析

ThreadLocalMap是ThreadLocal类中的静态内部类，可以看成一个map容器。Thread类中有一个全局变量threadLocals 就是ThreadLocalMap类型，它才是ThreadLocal存储数据的真正容器。

```java
public class Thread implements Runnable {
/* ThreadLocal values pertaining to this thread. This map is maintained
     * by the ThreadLocal class. */
    ThreadLocal.ThreadLocalMap threadLocals = null;
}
```

##### 4.1ThreadLocalMap底层结构

在看ThreadLocal的底层代码之前，我们先来看看ThreadLocalMap的操作，这样有利于接下来更好地理解。

```java
static class ThreadLocalMap {
    private static final int INITIAL_CAPACITY = 16;

    private int size = 0;

    private int threshold;

    private Entry[] table;    //Thread的map容器

    // 构造方法，在第一次调用ThreadLocal.set()时会new一个ThreadLocalMap
    ThreadLocalMap(ThreadLocal<?> firstKey, Object firstValue) {
        table = new Entry[INITIAL_CAPACITY];
        int i = firstKey.threadLocalHashCode & (INITIAL_CAPACITY - 1);
        table[i] = new Entry(firstKey, firstValue);
        size = 1;
        setThreshold(INITIAL_CAPACITY);
    }

    static class Entry extends WeakReference<ThreadLocal<?>> {
        Object value;
        Entry(ThreadLocal<?> k, Object v) {
            super(k);
            value = v;
        }
    }
}
```

可能会好奇，table中的Entry只有一个value，并没有像Hashmap中那样是键值对啊，为什么叫它map容器呢？其实Entry是有key的，key就是构造Entry时传入的ThreadLocal参数，不过指向ThreadLocal的成员变量是从Reference类中继承的。从Entry的构造方法中可看到传入了一个ThreadLocal对象，它就是key，我们追踪super(k)，最后发现在Entry的父类Reference中，传给了成员变量referent。至于这个referent是怎么用到容器的操作上，下面马上会看到。

```java
public abstract class Reference<T> {
    private T referent;           // key。注入了传入的ThreadLocal，被Entry继承

    //ThreadLocal最终传给了参数referent
    Reference(T referent, ReferenceQueue<? super T> queue) {
        this.referent = referent;
        this.queue = (queue == null) ? ReferenceQueue.NULL : queue;
    }

    // 获取“key”
    public T get() {
        return this.referent;
    }
}
```

##### 4.2ThreadLocalMap的set方法

set方法有点像HashMap的put方法，HashMap中也是用一个Node[] table来存储数据，Node里面包含了键值对，而这里Entry也是<referent,value>键值对。

首先根据根据key的hashcode计算出插入的Entry在table中的位置，然后从计算出的位置向前循环遍历，直到找到的Entry为null或key为null（被GC了），就把Entry插入到该位置。在遍历的过程中会判断遍历元素与插入元素的是否是相同的Entry，规则是比较Entry的key即referent。可以看到下面通过Entry.get()来获取key，而get()方法继承自Reference父类中，就是返回referent属性。

```java
    private void set(ThreadLocal<?> key, Object value) {
            Entry[] tab = table;
            int len = tab.length;
 
            //根据ThreadLocal的hash值计算应该从table中哪个位置开始(跟HashMap一样)
            int i = key.threadLocalHashCode & (len-1);
 
            //遍历table中的每一个Entry
            for (Entry e = tab[i];
                 e != null;
                 e = tab[i = nextIndex(i, len)]) {
                ThreadLocal<?> k = e.get();
 
                //如果找到相同的key，覆盖并返回
                if (k == key) {
                    e.value = value;
                    return;
                }
 
                //如果发现了空key为空，即ThreadLocal对象被GC了，就把数据放到该位置
                if (k == null) {
                    replaceStaleEntry(key, value, i);
                    return;
                }
            }
 
            //当返现e==null时，执行下面代码，new再检测是否扩容
            tab[i] = new Entry(key, value);
            int sz = ++size;
            if (!cleanSomeSlots(i, sz) && sz >= threshold)
                rehash();
        }
```

##### 4.3ThreadLocalMap的getEntry方法

首先根据key找到table中的一个位置，如果该位置上的元素不为空且key等于传入的key，则直接返回该位置上的Entry；若位置上的元素为null或key为null，则返回null；否则向前循环遍历table直到发现key相等的Entry返回，或遍历到null直接返回null。

```java
	private Entry getEntry(ThreadLocal<?> key) {
            int i = key.threadLocalHashCode & (table.length - 1);
            Entry e = table[i];
            if (e != null && e.get() == key)
                return e;
            else
                return getEntryAfterMiss(key, i, e);
        }
 
        private Entry getEntryAfterMiss(ThreadLocal<?> key, int i, Entry e) {
            Entry[] tab = table;
            int len = tab.length;
 
            while (e != null) {
                ThreadLocal<?> k = e.get();
                if (k == key)
                    return e;
                if (k == null)
                    expungeStaleEntry(i);
                else
                    i = nextIndex(i, len);
                e = tab[i];
            }
            return null;
        }
```

##### 4.4ThreadLocal的set方法

先拿到当前线程的ThreadLocalMap容器，若线程的容器为null（一次都没有使用过）则初始化，创建一个新的ThreadLocalMap同时将key和value传入，插入到指定位置，ThreadLocalMap的构造方法参见4.1中的代码。若不为空则直接调用该容器的set方法插入键值对。两种方法都是把自己当key传入。

```java
public void set(T value) {
        Thread t = Thread.currentThread();    //获取当前线程对象
        ThreadLocalMap map = getMap(t);       //拿到t的map容器
        if (map != null)        
            map.set(this, value);            //map不空，根据key(调用set方法的ThreadLocal对象)设置值
        else
            createMap(t, value);             //map为空则创建一个map(第一次插入，有点怪怪的)
    }
 
    // 从当前线程中获取ThreadLocalMap 
    ThreadLocalMap getMap(Thread t) {
        return t.threadLocals;
    }
 
    // 初始化当前线程的ThreadLocalMap容器
    void createMap(Thread t, T firstValue) {
        t.threadLocals = new ThreadLocalMap(this, firstValue);
    }
```

##### 4.5ThreadLocal的get方法

很简单，直接调用的ThreadLocalMap的get方法，把自己作为key传入getEntry方法中。若容器为null，则调用setInitialValue方法。

```java
public T get() {
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null) {
            ThreadLocalMap.Entry e = map.getEntry(this);
            if (e != null) {
                @SuppressWarnings("unchecked")
                T result = (T)e.value;
                return result;
            }
        }
        return setInitialValue();
    }
```

##### 4.6ThreadLocal的setInitialValue方法

会返回一个初始值，由initialValue返回。可以看到initialValue方法只是返回了一个null，如果我们不重写它的话。同时再加一个判断，若容器为null，则跟set方法一样进行初始化，不过这里的value不是传入的，而是initialValue，默认为null。

```java

private T setInitialValue() {
        T value = initialValue();
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null)
            map.set(this, value);
        else
            createMap(t, value);
        return value;
    }
 
protected T initialValue() {
        return null;
    }
```

#### 5.案例理解

下面创建了5个线程，在每个线程执行期间调用了两个ThreadLocal对象的set方法，然后打印get方法的返回值。显然同一个ThreadLocal在不同线程中调用get和set方法是能区分线程的。

```java
public class Test1 {
 
    public static void main(String[] args) {
        for (int i=0;i<5;++i){
            new MyThread().start();
        }
    }
}
 
class MyThread extends Thread{
 
    private static final ThreadLocal<String> threadLocal1 = new ThreadLocal<String>();
    private static final ThreadLocal<String> threadLocal2 = new ThreadLocal<String>();
 
    @Override
    public void run(){
        threadLocal1.set(getName()+"：调用了threadLocal1的set方法");
        threadLocal2.set(getName()+"：调用了threadLocal2的set方法");
        System.out.println(threadLocal1.get());
        System.out.println(threadLocal2.get());
    }
}
 
/*
---------------打印结果-----------------
Thread-0：调用了threadLocal1的set方法
Thread-0：调用了threadLocal2的set方法
Thread-2：调用了threadLocal1的set方法
Thread-2：调用了threadLocal2的set方法
Thread-3：调用了threadLocal1的set方法
Thread-4：调用了threadLocal1的set方法
Thread-3：调用了threadLocal2的set方法
Thread-4：调用了threadLocal2的set方法
Thread-1：调用了threadLocal1的set方法
Thread-1：调用了threadLocal2的set方法
*/
```

##### 5.1脏读

当使用线程池的时候，由于工作线程是循环利用的，上一个任务线程通过ThreadLocal在工作线程中存入了数据，下一个任务线程被该工作线程执行时，依然能够读到上一个任务线程存入的数据，也就是读到了脏数据。

解决办法就是在一个线程执行结束后将数据通过`ThreadLocal.remove()`删除掉。

##### 5.2内存泄漏

​	由于ThreadLocalMap是以弱引用的方式引用着ThreadLocal，换句话说，就是ThreadLocal是被ThreadLocalMap以弱引用的方式关联着，因此如果ThreadLocal没有被ThreadLocalMap以外的对象引用（如手动令ThreadLocal = null或下面红色字体），则在下一次GC的时候，ThreadLocal实例就会被回收，那么此时ThreadLocalMap里的一组KV的K就是null了，因此在没有额外操作的情况下，此处的V便不会被外部访问到，而且只要Thread实例一直存在，Thread实例就强引用着ThreadLocalMap，因此ThreadLocalMap就不会被回收，那么这里K为null的V就一直占用着内存。

综上，发生内存泄露的条件是

- ThreadLocal实例没有被外部强引用，比如我们假设在提交到线程池的task中实例化的ThreadLocal对象，当task结束时，ThreadLocal的强引用也就结束了
- ThreadLocal实例被回收，但是在ThreadLocalMap中的V没有被任何清理机制有效清理
- 当前Thread实例一直存在，则会一直强引用着ThreadLocalMap，也就是说ThreadLocalMap也不会被GC

​	也就是说，如果Thread实例还在，但是ThreadLocal实例却不在了，则ThreadLocal实例作为key所关联的value无法被外部访问，却还被强引用着，因此出现了内存泄露。

解决办法就是创建ThreadLocal时将ThreadLocal变量设置成全局static类型，当需要存储线程私有的数据时，通过全局的ThreadLocal变量来存，不要在普通方法体中定义局部的ThreadLocal变量来存数据；不要手动将ThreadLocal引用指向null。
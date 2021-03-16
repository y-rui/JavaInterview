### 集合类中的 List 和 Map 的线程安全版本是什么

Java 提供了不同层面的线程安全支持。在传统集合框架内部，除了 Hashtable 等同步容器，还提供了所谓的同步包装器（Synchronized Wrapper），我们可以调用 Collections 工具类提供的包装方法，来获取一个同步的包装容器（如 Collections.synchronizedMap），但是它们都是利用非常粗粒度的同步方式，在高并发情况下，性能比较低下。

另外，更加普遍的选择是利用并发包提供的线程安全容器类，它提供了：各种并发容器，比如 ConcurrentHashMap、CopyOnWriteArrayList。各种线程安全队列（Queue/Deque），如 ArrayBlockingQueue、SynchronousQueue。各种有序容器的线程安全版本等。

具体保证线程安全的方式，包括有从简单的 synchronize 方式，到基于更加精细化的，比如基于分离锁实现的 ConcurrentHashMap 等并发实现等。具体选择要看开发的场景需求，总体来说，并发包内提供的容器通用场景，远优于早期的简单同步实现。

### 如何保证线程安全的

#### List

1. 模拟多线程环境

多线程环境下，会抛出 java.util.ConcurrentModificationException 异常

```java
public static void listNotSafe() {
    List<String> list = new CopyOnWriteArrayList<>();

    for (int i = 0; i < 30; i++) {
        new Thread(() -> {
            list.add(UUID.randomUUID().toString().substring(0, 8));
            System.out.println(list);
        }).start();
    }
}
```

![1](/Users/yinrui/repository/JavaInterview/2021-03-16/1.png)

2. 异常原因

多线程环境下，并发争抢修改导致出现该异常。

3. 解决办法

```java
// 1. 使用线程安全类 Vector
new Vector();

// 2. 使用 Collections 工具类封装 ArrayList
Collections.synchronizedList(new ArrayList<>());

// 3. 使用 java.util.concurrent.CopyOnWriteArrayList;
new CopyOnWriteArrayList<>();
```

4. 写时复制思想

CopyOnWrite 容器即写时复制的容器。往一个容器添加元素的时候，不直接往当前容器Object[]添加，而是先将当前容器Object[]进行Copy, 复制出一个新的容器Object[] newElements， 然后新的容器Object[] newElements 里添加元素，添加完元素之后，再将原容器的引用指向新的容器 setArray(newElements); 这样做的好处是可以对CopyOnWrite容器进行并发的读，而不需要加锁，因为当前容器不会添加任何元素。所以CopyOnWrite容器也是一种读写分离的思想，读和写不同的容器。

```java
// CopyOnWriteArrayList.java
public boolean add(E e) {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        Object[] elements = getArray();
        int len = elements.length;
        Object[] newElements = Arrays.copyOf(elements, len + 1);
        newElements[len] = e;
        setArray(newElements);
        return true;
    } finally {
        lock.unlock();
    }
}
```

#### Set

1. 线程安全问题

与 List 接口的测试方法相似，同样会抛出 java.util.ConcurrentModificationException 异常。

2. 解决办法

```java
// 1. 使用 Collections 工具类封装
Collections.synchronizedSet(new HashSet<>());

// 2. 使用 java.util.concurrent.CopyOnWriteArraySet;
new CopyOnWriteArraySet<>();
```

3. CopyOnWriteArraySet

final ReentrantLock lock = this.lock; 为什么声明为 final？参考可以看看[这个](https://blog.csdn.net/zqz_zqz/article/details/79438502)

```java
// 底层实际上是一个 CopyOnWriteArrayList
public class CopyOnWriteArraySet<E> extends AbstractSet<E>
        implements java.io.Serializable {
    private static final long serialVersionUID = 5457747651344034263L;

    private final CopyOnWriteArrayList<E> al;

    // ...
}
```

```java
// 添加元素，相当于调用 CopyOnWriteArrayList 的 addIfAbsent() 方法
public class CopyOnWriteArraySet<E> {
    public boolean add(E e) {
        return al.addIfAbsent(e);
    }
}

/**
 * CopyOnWriteArrayList 的 addIfAbsent() 方法
 * Set 集合中的元素不可重复，如果原集合中有要添加的元素，则直接返回 false
 * 否则，将该元素加入集合中
 */
public boolean addIfAbsent(E e) {
    Object[] snapshot = getArray();
    return indexOf(e, snapshot, 0, snapshot.length) >= 0 ? false :
        addIfAbsent(e, snapshot);
}

/**
 * 重载的 addIfAbsent() 方法，用于真正添加元素加锁后，再次获取集合，与刚才拿到的集合比较，
 * 两次拿到的不一样，说明集合被其他线程修改过了，重新比较最新集合中有没有该元素，如果比较
 * 后，没有返回 false，说明没有该元素，执行下面的添加方法。
 */
private boolean addIfAbsent(E e, Object[] snapshot) {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        Object[] current = getArray();
        int len = current.length;
        if (snapshot != current) {
            // Optimize for lost race to another addXXX operation
            int common = Math.min(snapshot.length, len);
            for (int i = 0; i < common; i++)
                if (current[i] != snapshot[i] && eq(e, current[i]))
                    return false;
            if (indexOf(e, current, common, len) >= 0)
                return false;
        }
        Object[] newElements = Arrays.copyOf(current, len + 1);
        newElements[len] = e;
        setArray(newElements);
        return true;
    } finally {
        lock.unlock();
    }
}
```

#### Map

1. 线程安全问题

和上面一样，多线程环境下，会抛出 java.util.ConcurrentModificationException 异常。

2. 解决办法

```java
// 使用 Collections 工具类
Collections.synchronizedMap(new HashMap<>());

// 使用 ConcurrentHashMap
new ConcurrentHashMap<>();
```

3. HashMap、Hashtable 和 ConcurrentHashMap 的区别

**继承不同：**HashMap继承AbstractMap, Hashtable继承Dictonary，ConcurrentHashMap除了继承AbstractMap还实现了ConcurrentMap接口

**线程是否安全：**HashMap非线程安全，ConcurrentHashMap 和 Hashtable 线程安全，但是他们的实现机制不同，Hashtabl e使用synchronized实现同步方法，而ConcurrentHashMap降低锁的粒度，拥有更好的并发性能。

**Key-Value值：**ConcurrentHashMap和Hashtable都不允许value和key为null，但是HashMap允许唯一的key为null，和任意个value为null

**哈希算法不同：**HashMap 和 Jdk 8 中的 ConcurrentHashMap 的算法一致都是使用 key 的 hashcode 值进行高16位和低16位异或再取模长度，而Hashtable是直接对 key 的hashcode值进行取模操作 。

**扩容机制不同：**ConcurrentHashMap和HashMap的扩容机制和初始容量一致，扩容为原有数组长度的两倍，初始容量为16，但是hashtable中的初始容量为11，容量为原有长度的两倍+1。

**失败机制：**ConcurrentHashMap支持安全失败，HashMap和hashtable支持的快速失败

**查询方法：**HashMap没有contains方法，但是拥有containsKey和containsValue方法，Hashtable和ConcurrentHashMap还支持contains方法

**迭代方式：**ConcurrentHashMap和Hashtable还支持Enumeration迭代方式

### 快速失败迭代器（Fail-Fast Iterators）

在使用集合的时候，你也要了解到迭代器的并发策略：Fail-Fast Iterators
看下以后代码片段，遍历一个String类型的集合：

```java
List<String> listNames = Arrays.asList("Tom", "Joe", "Bill", "Dave", "John");
 
Iterator<String> iterator = listNames.iterator();
 
while (iterator.hasNext()) {
    String nextName = iterator.next();
    System.out.println(nextName);
}
```

这里我们使用了Iterator来遍历list中的元素，试想下listNames被两个线程共享：一个线程执行遍历操作，在还没有遍历完成的时候，第二线程进行修改集合操作（添加或者删除元素），你猜测下这时候会发生什么？
遍历集合的线程会立刻抛出异常“ConcurrentModificationException”，所以称之为：快速失败迭代器（随便翻的哈，没那么重要，理解就OK）
为什么迭代器会如此迅速的抛出异常？
因为当一个线程在遍历集合的时候，另一个在修改遍历集合的数据会非常的危险：集合可能在修改后，有更多元素了，或者减少了元素又或者一个元素都没有了。所以在考虑结果的时候，选择抛出异常。而且这应该尽可能早的被发现，这就是原因。（反正这个答案不是我想要的~）

下面这段代码演示了抛出：ConcurrentModificationException

```java
import java.util.*;
 
/**
 * This test program illustrates how a collection's iterator fails fast
 * and throw ConcurrentModificationException
 * @author www.codejava.net
 *
 */
public class IteratorFailFastTest {
 
    private List<Integer> list = new ArrayList<>();
 
    public IteratorFailFastTest() {
        for (int i = 0; i < 10_000; i++) {
            list.add(i);
        }
    }
 
    public void runUpdateThread() {
        Thread thread1 = new Thread(new Runnable() {
 
            public void run() {
                for (int i = 10_000; i < 20_000; i++) {
                    list.add(i);
                }
            }
        });
 
        thread1.start();
    }
 
 
    public void runIteratorThread() {
        Thread thread2 = new Thread(new Runnable() {
 
            public void run() {
                ListIterator<Integer> iterator = list.listIterator();
                while (iterator.hasNext()) {
                    Integer number = iterator.next();
                    System.out.println(number);
                }
            }
        });
 
        thread2.start();
    }
 
    public static void main(String[] args) {
        IteratorFailFastTest tester = new IteratorFailFastTest();
 
        tester.runIteratorThread();
        tester.runUpdateThread();
    }
}
```

如你所见，在thread1遍历list的时候，thread2执行了添加元素的操作，这时候异常被抛出。
需要注意的是，使用iterator遍历list，快速失败的行为是为了让我更早的定位问题所在。我们不应该依赖这个来捕获异常，因为快速失败的行为是没有保障的。这意味着如果抛出异常了，程序应该立刻终止行为而不是继续执行。
现在你应该了解到了ConcurrentModificationException是如何工作的，而且最好是避免它。

### 同步封装器

至此我们明白了，为了确保在单线程环境下的性能最大化，所以基础的集合实现类都没有保证线程安全。那么如果我们在多线程环境下如何使用集合呢？
当然我们不能使用线程不安全的集合在多线程环境下，这样做会导致出现我们期望的结果。我们可以手动自己添加synchronized代码块来确保安全，但是使用自动线程安全的线程比我们手动更为明智。
你应该已经知道，Java集合框架提供了工厂方法创建线程安全的集合，这些方法的格式如下：

```java
Collections.synchronizedXXX(collection)
```

这个工厂方法封装了指定的集合并返回了一个线程安全的集合。XXX可以是Collection、List、Map、Set、SortedMap和SortedSet的实现类。比如下面这段代码创建了一个线程安全的列表：

```java
List<String> safeList = Collections.synchronizedList(new ArrayList<>());
```

如果我们已经拥有了一个线程不安全的集合，我们可以通过以下方法来封装成线程安全的集合：

```java
Map<Integer, String> unsafeMap = new HashMap<>();
Map<Integer, String> safeMap = Collections.synchronizedMap(unsafeMap);
```

如你锁看到的，工厂方法封装指定的集合，返回一个线程安全的结合。事实上接口基本都一直，只是实现上添加了synchronized来实现。所以被称之为：同步封装器。后面集合的工作都是由这个封装类来实现。

提示：
在我们使用iterator来遍历线程安全的集合对象的时候，我们还是需要添加synchronized字段来确保线程安全，因为Iterator本身并不是线程安全的，请看代码如下：

```java
List<String> safeList = Collections.synchronizedList(new ArrayList<>());
 
// adds some elements to the list
 
Iterator<String> iterator = safeList.iterator();
 
while (iterator.hasNext()) {
    String next = iterator.next();
    System.out.println(next);
}
```

事实上我们应该这样来操作：

```java
synchronized (safeList) {
    while (iterator.hasNext()) {
        String next = iterator.next();
        System.out.println(next);
    }
}
```

同时提醒下，Iterators也是支持快速失败的。
尽管经过类的封装可保证线程安全，但是他们依然有着自己的缺点，具体见下面部分。

### 并发集合

一个关于同步集合的缺点是，用集合的本身作为锁的对象。这意味着，在你遍历对象的时候，这个对象的其他方法已经被锁住，导致其他的线程必须等待。其他的线程无法操作当前这个被锁的集合，只有当执行的线程释放了锁。这会导致开销和性能较低。
这就是为什么jdk1.5+以后提供了并发集合的原因，因为这样的集合性能更高。并发集合类并放在java.util.concurrent包下，根据三种安全机制被放在三个组中。

- 第一种为：写时复制集合：这种集合将数据放在一成不变的数组中；任何数据的改变，都会重新创建一个新的数组来记录值。这种集合被设计用在，读的操作远远大于写操作的情景下。有两个如下的实现类：CopyOnWriteArrayList 和 CopyOnWriteArraySet.
  需要注意的是，写时复制集合不会抛出ConcurrentModificationException异常。因为这些集合是由不可变数组支持的，Iterator遍历值是从不可变数组中出来的，不用担心被其他线程修改了数据。
- 第二种为：比对交换集合也称之为CAS（Compare-And-Swap）集合：这组线程安全的集合是通过CAS算法实现的。CAS的算法可以这样理解：
  为了执行计算和更新变量，在本地拷贝一份变量，然后不通过获取访问来执行计算。当准备好去更新变量的时候，他会跟他之前的开始的值进行比较，如果一样，则更新值。
  如果不一样，则说明应该有其他的线程已经修改了数据。在这种情况下，CAS线程可以重新执行下计算的值，更新或者放弃。使用CAS算法的集合有：ConcurrentLinkedQueue and ConcurrentSkipListMap.
  需要注意的是，CAS集合具有不连贯的iterators，这意味着自他们创建之后并不是所有的改变都是从新的数组中来。同时他也不会抛出ConcurrentModificationException异常。
- 第三种为：这种集合采用了特殊的对象锁（java.util.concurrent.lock.Lock）：这种机制相对于传统的来说更为灵活，可以如下理解：
  这种锁和经典锁一样具有基本的功能，但还可以再特殊的情况下获取：如果当前没有被锁、超时、线程没有被打断。
  不同于synchronization的代码,当方法在执行，Lock锁一直会被持有，直到调用unlock方法。有些实现通过这种机制把集合分为好几个部分来提供并发性能。比如：LinkedBlockingQueue，在队列的开后和结尾，所以在添加和删除的时候可以同时进行。
  其他使用了这种机制的集合有：ConcurrentHashMap 和绝多数实现了BlockingQueue的实现类
  同样的这一类的集合也具有不连贯的iterators，也不会抛出ConcurrentModificationException异常。


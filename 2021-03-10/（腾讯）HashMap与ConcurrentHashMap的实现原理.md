### （腾讯）HashMap 与 ConcurrentHashMap 的实现原理是怎样的？ConcurrentHashMap 是如何保证线程安全的？

#### HashMap实现原理

HashMap的底层主要是基于数组和链表来实现的，Java1.8后数组大小超过64并且链表中元素个数超过8个就将链表转换为红黑树

#### ConcurrentHashMap实现原理

将数据分段每一段配一把锁，从而提高并发效率，Java1.7中采用Segment+HashEntry的方式实现，Java1.8取消Segment分段锁的数据结构，取而代之的是对每个数组元素加锁，采用的是Node+CAS+Synchronized保证并发安全
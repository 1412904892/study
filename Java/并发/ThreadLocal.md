## 两个问题

### 什么是TheadLocal？  
    ThreadLocal类顾名思义可以理解为线程本地变量。也就是说如果定义了一个ThreadLocal，每个线程往这个ThreadLocal中读写是线程隔离，互相之间不会影响的。它提供了一种将可变数据通过每个线程有自己的独立副本从而实现线程封闭的机制。  

### 实现思路  
    Thread类有一个类型为ThreadLocal.ThreadLocalMap的实例变量threadLocals，也就是说每个线程有一个自己的ThreadLocalMap。ThreadLocalMap有自己的独立实现，可以简单地将它的key视作ThreadLocal，value为代码中放入的值（实际上key并不是ThreadLocal本身，而是它的一个弱引用）。每个线程在往某个ThreadLocal里塞值的时候，都会往自己的ThreadLocalMap里存，读也是以某个ThreadLocal作为引用，在自己的map里找对应的key，从而实现了线程隔离。

## 为什么要用弱引用？  
如果使用普通的key-value的形式来定义存储结构，实质上会造成节点的生命周期与线程强绑定，只要线程没有销毁，那么在GC分析中一直处于可达状态，没有办法回收，而程序本身无法判断是否可以清理节点。弱引用是Java中四档引用的第三档，比软引用更加弱一些，如果一个对象没有强引用链可达，那么一般活不过下一次GC。当某个ThreadLocal已经没有强引用可达，则随着它被垃圾回收，在ThreadLocalMap里对应的Entry的键值会失效，这为ThreadLocalMap本身的垃圾清理提供了便利。

## ThreadLocal与内存泄漏
如果一个ThreadLocal对象被回收了，但是value对于[当前线程->当前线程ThreadLocals->Entry->某个entry.value]这样的一条强引用链是可达的，因此value不会被回收。

## getEntry方法
这个方法会被ThreadLocal的get方法调用，用于获得map中某个ThreadLocal存放的value。  
```Java
private Entry getEntry(ThreadLocal<?> key) {
    // 根据key这个ThreadLocal的ID来获取索引，也即哈希值
    int i = key.threadLocalHashCode & (table.length - 1);
    Entry e = table[i];
    // 对应的entry存在且未失效且弱引用指向的ThreadLocal就是key，则命中返回
    if (e != null && e.get() == key) {
        return e;
    } else {
        // 因为用的是线性探测，所以往后找还是有可能能够找到目标Entry的。
        return getEntryAfterMiss(key, i, e);
    }
}

/*
 * 调用getEntry未直接命中的时候调用此方法
 */
private Entry getEntryAfterMiss(ThreadLocal<?> key, int i, Entry e) {
    Entry[] tab = table;
    int len = tab.length;
   
    
    // 基于线性探测法不断向后探测直到遇到空entry。
    while (e != null) {
        ThreadLocal<?> k = e.get();
        // 找到目标
        if (k == key) {
            return e;
        }
        if (k == null) {
            // 该entry对应的ThreadLocal已经被回收，调用expungeStaleEntry来清理无效的entry
            expungeStaleEntry(i);
        } else {
            // 环形意义下往后面走
            i = nextIndex(i, len);
        }
        e = tab[i];
    }
    return null;
}

/**
 * 这个函数是ThreadLocal中核心清理函数，它做的事情很简单：
 * 就是从staleSlot开始遍历，将无效（弱引用指向对象被回收）清理，即对应entry中的value置为null，将指向这个entry的table[i]置为null，直到扫到空entry。
 * 另外，在过程中还会对非空的entry作rehash。
 * 可以说这个函数的作用就是从staleSlot开始清理连续段中的slot（断开强引用，rehash slot等）
 */
private int expungeStaleEntry(int staleSlot) {
    Entry[] tab = table;
    int len = tab.length;

    // 因为entry对应的ThreadLocal已经被回收，value设为null，显式断开强引用
    tab[staleSlot].value = null;
    // 显式设置该entry为null，以便垃圾回收
    tab[staleSlot] = null;
    size--;

    Entry e;
    int i;
    for (i = nextIndex(staleSlot, len); (e = tab[i]) != null; i = nextIndex(i, len)) {
        ThreadLocal<?> k = e.get();
        // 清理对应ThreadLocal已经被回收的entry
        if (k == null) {
            e.value = null;
            tab[i] = null;
            size--;
        } else {
            /*
             * 对于还没有被回收的情况，需要做一次rehash。
             * 
             * 如果对应的ThreadLocal的ID对len取模出来的索引h不为当前位置i，
             * 则从h向后线性探测到第一个空的slot，把当前的entry给挪过去。
             */
            int h = k.threadLocalHashCode & (len - 1);
            if (h != i) {
                tab[i] = null;
                
                /*
                 * 在原代码的这里有句注释值得一提，原注释如下：
                 *
                 * Unlike Knuth 6.4 Algorithm R, we must scan until
                 * null because multiple entries could have been stale.
                 *
                 * 这段话提及了Knuth高德纳的著作TAOCP（《计算机程序设计艺术》）的6.4章节（散列）
                 * 中的R算法。R算法描述了如何从使用线性探测的散列表中删除一个元素。
                 * R算法维护了一个上次删除元素的index，当在非空连续段中扫到某个entry的哈希值取模后的索引
                 * 还没有遍历到时，会将该entry挪到index那个位置，并更新当前位置为新的index，
                 * 继续向后扫描直到遇到空的entry。
                 *
                 * ThreadLocalMap因为使用了弱引用，所以其实每个slot的状态有三种也即
                 * 有效（value未回收），无效（value已回收），空（entry==null）。
                 * 正是因为ThreadLocalMap的entry有三种状态，所以不能完全套高德纳原书的R算法。
                 *
                 * 因为expungeStaleEntry函数在扫描过程中还会对无效slot清理将之转为空slot，
                 * 如果直接套用R算法，可能会出现具有相同哈希值的entry之间断开（中间有空entry）。
                 */
                while (tab[h] != null) {
                    h = nextIndex(h, len);
                }
                tab[h] = e;
            }
        }
    }
    // 返回staleSlot之后第一个空的slot索引
    return i;
}
```
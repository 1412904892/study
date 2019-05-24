## volatile特性
1. volatile具有可见性，即线程A对一个变量的写入，其他线程线程可以立即看到
2. volatile具有有序性，volatile关键字可以禁止指令重排序
   
在《深入理解Java虚拟机》一种，有这样的一段话，比较加入了volatile关键字和没有加入volatile关键字的汇编代码可以看见，加入volatile关键字时会多出一个lock前缀指令。
lock前缀指令相当于一个内存屏障，内存屏障有以下三个功能：
1）内存屏障可以确保指令在重排序时，其后面的指令不会重排序到内存屏障之前，其前面的指令不会重排序到内存屏障之后。
2）内存屏障会强制将对缓存的修改操作立即写入主存中
3）如果是写操作，会使其他CPU中缓存行无效


```Java

 final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                   boolean evict) {
        Node<K,V>[] tab; Node<K,V> p; int n, i;
        if ((tab = table) == null || (n = tab.length) == 0)
        	// table 为空时， 通过 resize() 方法进行初始化
            n = (tab = resize()).length;
        if ((p = tab[i = (n - 1) & hash]) == null)// 注意到此处指针 p 已经被指向了桶中的一个元素
        	// 此处通过（n - 1） & hash 计算出该元素在桶数组中的下标， 如果此位置为空，则可以直接放置该元素
        	// 为什么通过 (n - 1) & hash 计算下标在文章后面详细解释
            tab[i] = newNode(hash, key, value, null);
        else {
        	// 下面对应桶的位置已经被占用的情况， 属于 hash 取模后索引·冲突解决的部分
            Node<K,V> e; K k;// 初始化 element 指针 e， 如果当前待插入的 key 值经过后续的搜索后， 发现已经存在， 该指针会已经存在的元素位置， 否则为空
            if (p.hash == hash && // 桶中已经放置的元素hash值是否和当前待放置的元素hash值相等
                ((k = p.key) == key || (key != null && key.equals(k))))// 且桶中已经放置的元素 key 值和当前待放置的元素 key 值相同
                e = p; // 指向已经存在的元素位置 
            else if (p instanceof TreeNode)
            // 如果桶中已经放置的元素是一个树节点，说明这个桶的位置上已经发生多次冲突， 属于这个位置的多个元素以自平衡二叉树的结构, 连接在这个桶的后面了所以新的待放置的元素需要插入到这颗树中，故调用 putTreeVal
            // 此处传入当前 hashMap 的引用 this 的原因是， putTreeVal() 是一个定义在静态内部类 TreeNode 的方法， 该方法内部需要调用一个定义在 HashMap 类的非静态方法 newTreeNode() ， 而静态内部类是不能直接访问外部类的非静态成员的， 所以需要传入引用
                e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value)    
            else {
            // 桶的位置上是一个链表头
                for (int binCount = 0; ; ++binCount) {// binCount 用于计数链表中的元素个数
                    if ((e = p.next) == null) {// p.next 为空说明到达链表尾
                        p.next = newNode(hash, key, value, null);// 尾部插入当前待放置的元素
                        if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                        // 插入成功后， 判断链表长度是否大于阈值， 链表过长需要转化成树的结构，加速检索效率 
                            treeifyBin(tab, hash);
                        break;// 跳出循环
                    }
                    if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))
                        // 如果在遍历的过程中发现链表中已经存在该 key 值相同的元素，跳出循环 
                        break;
                    p = e;
                }
            }
            if (e != null) { // existing mapping for key
            // 这个地方针对桶中或桶后链表中发现key值相同元素的情形
            // 根据onlyIfAbsent 参数决定是否对已有元素的值进行替换
                V oldValue = e.value;
                if (!onlyIfAbsent || oldValue == null)
                    e.value = value;
                afterNodeAccess(e);// 用于LinkedHashMap 的方法， 对于HashMap无意义
                return oldValue;
            }
        }
        ++modCount; //HashMap的数据被修改的次数，这个变量用于迭代过程中的Fail-Fast机制，其存在的意义在于保证发生了线程安全问题时，能及时的发现（操作前备份的count和当前modCount不相等）并抛出异常终止操作。
        if (++size > threshold)// hashMap 节点数目大于阈值， 进行扩容
            resize();
        afterNodeInsertion(evict);// 用于LinkedHashMap 的方法， 对于HashMap无意义
        return null;
    }
--------------------- 
作者：萧萧冷 
来源：CSDN 
原文：https://blog.csdn.net/lengxiao1993/article/details/84029155 
版权声明：本文为博主原创文章，转载请附上博文链接！

```
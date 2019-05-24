ReentrantLock类中实现了一个抽象类Sync，ReentrantLock根据构造方法的布尔型参数判断出Sync的实现类FairSync和NonfairSync。  

当我们定义初始化一个ReentrantLock时，并使用lock.lock()  
如果这个线程是第一个出现的，那么它的调用链如下:  
lock.lock()---->compareAndSetState(0,1)---->setExclusiveOwnerThread(Thread.currentThread())
当一个线程首次获取时，会首先利用CAS去判断state的状态，如果此时的state=0表示没有线程占有锁，那么会将利用CAS操作将其修改为1，并将设置当前线程为获取独占锁的线程。

此时如果有第二线程执行到lock.lock()，那么compareAndSetState(0,1)这个操作肯定不会成功，此时会调用acquire(1)。整个调用链为：  


首先进入到了lock函数之后，会先尝试利用CAS判断state的值是否为0，如果为0，就成为了第一步的操作，这里假设线程1还没有释放锁，那么这一步的操作必然失败，转而执行acquire(1).  
```Java

```
在acquire()函数中，调用依次调用了三个函数，tryacquire()函数，这个函数在AQS中没有实现，需要实现同步器或者锁的用户自己实现，这时一个模板方法，在ReentrantLock中由于存在公平锁和非公平锁，这个主要讲的是非公平锁，在tryacquire()中有调用了自己实现的nonfairTryAcquire()函数，在这个函数中，首先会拿到最新的state的值，这是因为state是volatitle修饰的，在这个调用的期间可能线程1已经释放了锁，所以在这里再次判断了能否持有锁，如果不可以返回false，继续执行下面的代码。然后判断当前线程是否是重入的，如果是重入的，那么将state的值继续向上累加，可重入的次数为Integet.MAX，如果不是上述的两种情况，就会返回false，然后在acquire中执行下一个函数。


addWaiter()是在AQS中实现的，作用是将当前的线程整理成一个Node节点，并加入到等待队列的队尾。然后会返回一个node。表示当前线程的在等待队列中的信息节点。这个node作为参数传入到acquireQueued()中执行。这个函数中是一个死循环，进入到这个函数中后，会先判断这个节点的前驱节点是否为头结点，如果是头结点，表示这个节点可以再次尝试获取锁，如果还是获取锁失败，则先调用AQS中的shouldParkAfterFailedAcquire()方法，判断这个线程能都挂起，然后执行parkAndCheckInterrupt()将线程阻塞。


https://www.cnblogs.com/xrq730/p/4979021.html

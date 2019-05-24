## synchronized和ReentrantLock之间的区别
1. 两者都是可重入锁，所谓的可重入锁就是当前线程可以再次获得自己的内部锁。同一个线程每次获取锁，锁的计数器都会加1，当计数器为0时这个锁才能释放
2. synchronized依赖于JVM，之前对synchronized的优化都是基于JVM的，而ReentrantLock是基于JDK的实现的，
3. ReentrantLock增加的新功能
   - 等待可中断：ReentrantLock提供了一种能够中断等待锁的线程的机制，也就是说正在等待的线程可以选择放弃等待，转而处理其他的事情
   - ReentrantLock可以指定这个锁是公平锁还是非公平锁，而synchronized只能是非公平锁
   - ReentrantLock可实现选择性通知，synchronized关键字主要是与wait()和notify()/notifyAll()相结合来实现等待/通知机制，ReentrantLock提供了条件Condition，对线程的等待/唤醒操作更加灵活。所以ReentrantLock更适合在多个条件变量和高度竞争锁的地方。
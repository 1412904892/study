- [说一说自己对于 synchronized 关键字的了解](#%E8%AF%B4%E4%B8%80%E8%AF%B4%E8%87%AA%E5%B7%B1%E5%AF%B9%E4%BA%8E-synchronized-%E5%85%B3%E9%94%AE%E5%AD%97%E7%9A%84%E4%BA%86%E8%A7%A3)
- [怎么使用synchronized关键字](#%E6%80%8E%E4%B9%88%E4%BD%BF%E7%94%A8synchronized%E5%85%B3%E9%94%AE%E5%AD%97)
  - [双重校验锁实现对象单例](#%E5%8F%8C%E9%87%8D%E6%A0%A1%E9%AA%8C%E9%94%81%E5%AE%9E%E7%8E%B0%E5%AF%B9%E8%B1%A1%E5%8D%95%E4%BE%8B)
- [synchronized的底层原理](#synchronized%E7%9A%84%E5%BA%95%E5%B1%82%E5%8E%9F%E7%90%86)
- [JDK1.6 之后的synchronized 关键字底层做了哪些优化](#jdk16-%E4%B9%8B%E5%90%8E%E7%9A%84synchronized-%E5%85%B3%E9%94%AE%E5%AD%97%E5%BA%95%E5%B1%82%E5%81%9A%E4%BA%86%E5%93%AA%E4%BA%9B%E4%BC%98%E5%8C%96)
  - [偏向锁](#%E5%81%8F%E5%90%91%E9%94%81)
  - [轻量级锁](#%E8%BD%BB%E9%87%8F%E7%BA%A7%E9%94%81)
  - [自旋锁和自适应自旋](#%E8%87%AA%E6%97%8B%E9%94%81%E5%92%8C%E8%87%AA%E9%80%82%E5%BA%94%E8%87%AA%E6%97%8B)
  - [锁消除](#%E9%94%81%E6%B6%88%E9%99%A4)
  - [锁粗化](#%E9%94%81%E7%B2%97%E5%8C%96)


## 说一说自己对于 synchronized 关键字的了解
synchronized关键字解决的是多个线程之间访问资源的同步性，synchronized关键字可以保证被他修饰的方法或者代码块在任意时刻只有一个线程执行。

在Java的早起版本中，synchronized属于重量级锁，效率低下，因为监视器锁是依赖于底层的操作系统的mutex Lock来实现的，Java的线程是映射到操作系统的原生线程之上的。如果要挂起或者唤醒一个线程，都需要操作系统帮忙完成，而操作系统实现线程之间的切换需要从用户态转换到内核态，这个状态之间的转换需要相对较长的时间，这也是为什么早期synchronized效率低的原因。在Java 6之后Java官方从JVM层面对其进行了优化。

## 怎么使用synchronized关键字
synchronized关键字最主要的使用方式：  
1）修饰实例方法，作用于当前对象实例加锁，进入同步代码前要获得当前对象实例的锁。  

2）修饰静态方法，作用于当前类对象加锁，进入同步代码前要获得当前类对象的锁。也就是要给当前类加锁，会作用于类的所有对象实例，因为静态成员不属于任何一个实例对象，是类成员（static表明这时该类的静态资源，不管new多少个对象，只有一份，所以对该类的所有对象都加了锁）。所以如果线程A调用一个实例对象的非静态synchronized方法，而线程B调用这个实例对象所属类的静态synchronized方法，是允许的，不会发生互斥现象，因为访问静态synchronized方法占用的锁是当前类的锁，而访问非静态synchronized方法占用的锁是当前实例对象锁。  

3）修饰代码块，指定加锁对象，对给定的对象加锁，进入同步代码库钱要获得给定对象的锁，和 synchronized 方法一样，synchronized(this)代码块也是锁定当前对象的。synchronized 关键字加到 static 静态方法和 synchronized(class)代码块上都是是给 Class 类上锁。这里再提一下：synchronized关键字加到非 static 静态方法上是给对象实例上锁。另外需要注意的是：尽量不要使用 synchronized(String a) 因为JVM中，字符串常量池具有缓冲功能！

### 双重校验锁实现对象单例

```Java
public class Singleton {

    private volatile static Singleton uniqueInstance;

    private Singleton() {
    }

    public static Singleton getUniqueInstance() {
       //先判断对象是否已经实例过，没有实例化过才进入加锁代码
        if (uniqueInstance == null) {
            //类对象加锁
            synchronized (Singleton.class) {
                if (uniqueInstance == null) {
                    uniqueInstance = new Singleton();
                }
            }
        }
        return uniqueInstance;
    }
}
```
这里为什么使用了volatile？
uniqueInstance 采用 volatile 关键字修饰也是很有必要的， uniqueInstance = new Singleton(); 这段代码其实是分为三步执行：
1）为 uniqueInstance 分配内存空间
2）初始化 uniqueInstance
3）将 uniqueInstance 指向分配的内存地址
但是因为JVM的指令重排的特性，执行顺序可能变为1->3->2。在多线程的环境下会导致一个线程获得还没有初始化的实例。例如，线程T1执行了1和3，此时T2调用getUniqueInstance（）后发现，uniqueInstance不为空，因此返回uniqueInstance，但是此时它还没有被初始化。
使用volatile可以禁止指令重排，保证在多线程的环境也能正常运行。

## synchronized的底层原理
1. synchronized 同步语句块的情况
```java
public class SynchronizedDemo {
	public void method() {
		synchronized (this) {
			System.out.println("synchronized 代码块");
		}
	}
}
```

synchronized同步语句块的实现使用的monitorenter和monitorexit指令，其中monitorenter指令值同步代码块的开始位置，monitorexit指明同步代码块的结束位置。当执行monitorenter指令时，线程试图获取锁也就是获取monitor（monitor存在于每个对象的对象头中，synchronized锁便是通过这种方式获取锁的，也就是为什么Java中任意对象可以作为锁的原因）的持有权，当计数器为0时则可以成功获取，获取后将锁计数器加1.相应的在执行monitorexit指令后，将计数器设为0，表明锁被释放，如果获取对象锁失败，那么当前线程就阻塞等待，知道锁被另外线程释放。  

2. synchronized修饰方法
```Java
public class SynchronizedDemo2 {
	public synchronized void method() {
		System.out.println("synchronized 方法");
	}
}
```
synchronized 修饰的方法并没有 monitorenter 指令和 monitorexit 指令，取得代之的确实是 ACC_SYNCHRONIZED 标识，该标识指明了该方法是一个同步方法，JVM 通过该 ACC_SYNCHRONIZED 访问标志来辨别一个方法是否声明为同步方法，从而执行相应的同步调用。

## JDK1.6 之后的synchronized 关键字底层做了哪些优化
JDK1.6 对锁的实现引入了大量的优化，如偏向锁、轻量级锁、自旋锁、适应性自旋锁、锁消除、锁粗化等技术来减少锁操作的开销。
### 偏向锁  
引入偏向锁的目的和引入轻量级锁的目的很像，都是为了在没有多线程竞争的前提下，减少传统的重量级锁使用操作系统互斥量产生的性能消耗。但是不同之处在于：轻量级锁在无竞争的情况下使用CAS操作去代替使用互斥量，而偏向锁在无竞争的情况下把整个同步都消除掉。
偏向锁的“偏”就是偏心的意思，他的意思是会偏向第一个获得它的线程，如果接下来的执行中，该锁没有被其他线程获取，那么持有偏向锁的线程就不需要进行同步！
但是对于竞争激烈的场合，偏向锁就会失效，因为这种情况下有可能每次申请锁的线程都是不用的，因此在这种情况下最好不要使用偏向锁，否则会得不偿失。偏向锁失败够，并不会立即膨胀为重量级锁，而是先升级为轻量级锁。
### 轻量级锁  
倘若偏向锁失败，虚拟机不会立即升级为重量级锁，他还会尝试使用一种被称为轻量级锁的优化手段。轻量级锁不是为了代替重量级锁，他的本意是在没有多线程竞争的前提下，减少传统的重量级锁使用操作系统互斥量产生的性能消耗，因为使用轻量级锁时，不需要申请互斥量。另外，轻量级锁的加锁和解锁都用到了CAS操作。

轻量级锁能够提升程序同步性能的依据是“对于绝大部分锁，在整个同步周期内都是不存在竞争的”，这时一个经验数据。如果没有竞争，轻量级锁使用CAS操作避免了使用互斥操作的开销。但是如果存在锁竞争，除了互斥量开销外，还会额外发生CAS操作，因此在有锁竞争的情况下，轻量级锁比传统的重量级锁更慢。如果锁竞争激励，那么轻量级将很快膨胀为重量级锁。
### 自旋锁和自适应自旋  
轻量级锁失败后，虚拟机为了避免线程真实的在操作系统层面挂起，还会进行一项被称为自旋锁的优化手段。
互斥同步对性能最大的影响就是阻塞实现，因为挂起线程/恢复线程的操作都需要转入内核态完成。
一般线程持有锁的时间不会太长，所以仅仅为了这一点时间去挂起线程/恢复线程是得不偿失的。所以，为了让一个线程等待，只需要让线程执行一个忙循环，这就是自旋。虚拟机自选次数默认为10次。另外,在 JDK1.6 中引入了自适应的自旋锁。自适应的自旋锁带来的改进就是：自旋的时间不在固定了，而是和前一次同一个锁上的自旋时间以及锁的拥有者的状态来决定，虚拟机变得越来越“聪明”了。

### 锁消除
锁消除理解起来很简单，他值得就是虚拟机即使编译器在运行时，如果检测到那些共享数据不可能存在竞争，那么就执行锁消除。锁消除节省了毫无意义的请求锁时间。

### 锁粗化
通常情况下，为了保证多线程间的有效并发，会要求每个线程持有锁的时间尽可能短，但是大某些情况下，一个程序对同一个锁不间断、高频地请求、同步与释放，会消耗掉一定的系统资源，因为锁的讲求、同步与释放本身会带来性能损耗，这样高频的锁请求就反而不利于系统性能的优化了，虽然单次同步操作的时间可能很短。锁粗化就是告诉我们任何事情都有个度，有些情况下我们反而希望把很多次锁的请求合并成一个请求，以降低短时间内大量锁请求、同步、释放带来的性能损耗。
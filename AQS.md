 # AQS源码

所谓的`AQS`就是**AbstractQueuedSynchronizer**，抽象队列同步器，是位于`JUC`包下面的

## 官方文档

> Provides a framework for implementing blocking locks and related synchronizers (semaphores, events, etc) that rely on first-in-first-out (FIFO) wait queues.  
> 
> This class is designed to be a useful basis for most kinds of synchronizers that rely on a single atomic {@code int} value to represent state.
>
> Subclasses must define the protected methods that change this state, and which define what that state means in terms of this object being acquired or released.  
> 
> Given these, the other methods in this class carry out all queuing and blocking mechanics. 
>
>  Subclasses can maintain other state fields, but only the atomically updated {@code int} value manipulated using methods {@link #getState}, {@link #setState} and {@link #compareAndSetState} is tracked with respect to synchronization.

大致的意思就是说：提供了一个依赖于先进先出的阻塞队列实现的阻塞锁还有相关同步器的框架。这个类设计的初衷就是在为大多数依赖单个原子{int类型}值表示状态的同步器提供有用的基础。子类必须定义`protected`类型的来改变这个状态的方法，并定义该状态对于获取或释放此对象而言意味着什么。

那上面的算是一个大概了，下面的才是底层的原理到底是什么样的。

>The wait queue is a variant of a "CLH" (Craig, Landin, and Hagersten) lock queue.  CLH locks are normally used for spinlocks.  
>
>We instead use them for blocking synchronizers, but use the same basic tactic(战术) of holding some of the control information about a thread in the predecessor of its node.
>
>  A "status" field in each node keeps track of whether a thread should block.  A node is signalled when its predecessor releases.  
>
>Each node of the queue otherwise serves as a specific-notification-style monitor holding a single waiting thread. 
>
>The status field does NOT control whether threads are granted locks etc though.  A thread may try to acquire if it is first in the queue. But being first does not guarantee success;
>
> it only gives the right to contend.  So the currently released contender thread may need to rewait.

翻译一下大概的意思：等待队列是一种`CLH`锁的一种变体，`CLH`锁通常用来作为一种自旋锁。我们将它们用于阻塞同步器，但是使用相同的基本策略，将有关线程的某些控制信息保存在其节点的前面的节点中。每个节点使用一个`status`字段来跟踪这个节点是否应该被堵塞，当前面的节点释放资源之后，后面的节点会被唤醒。队列的每个节点都作为一个特殊的通知的监视器。

## 说人话

所谓的AQS，AQS提供了一个先进先出的队列(FIFO)，用来实现同步锁。也就是说这个类里面维护了一个FIFO的线程等待队列，当然除此之外还维护了一个`volatile state`

这里再简单的说一下`volatile`。不想看的可以直接跳到[这里](#回到AQS)

### volatile

volatile是一个java的**关键字**，具有下面三个特性：

+ 线程之间可见性
+ 不保证原子性
+ 禁止指令重新排列

一个一个来说吧。

#### 线程之间可见性

每个线程访问一个变量都是从主内存拷贝一份到各自线程的工作空间的。举个例子，如果有3个线程，想要修改一个变量**a**的值，那修改的过程就是三个线程分别拷贝一份**a**到自己的工作空间，修改完成之后，再将这个值放回主内存。那就由一个问题了，如果三个线程对这个**a**都不可见，那一个线程修改了，别的线程怎么知道修改了呢？简单的写一个demo来看一下，下面的代码纯手打，可能会出错，不过肯定会看得懂的。

```java
public class Test{
    private static int a = 0;
    
    public void static main(String[] args){
        new Thread(()->{
            while(a==0){
                
            }
        },"Thread A").start();
        a=1;
    }
}
```

这段代码开启了两个线程，第一个线程当a=0的时候一直在循环，主线程做的事是当a=0的时候将a改为1。

如果主线程的修改对线程A不可见的话，那么线程A是会一直空循环的。

要解决上面的问题，就可以在定义a的时候加上`volatile`，保证a的修改是透明的就行。

#### 不保证原子性

原子性，这个词在数据库的`ACID`中出现了，就是一个事务要么成功，要么失败。这里也是一样的，一个线程的操作要么一起成功要么失败。但是`volatile`是不会保证这个特性的，还是举一个例子来说。代码还是手打的:joy:

```java
public class Test{
    private volatile static int a = 0;
    public void static main(String[] args){
        for(int i=0;i<10;i++){
            new Thread(()->{
                for(int j=0;j<1000;j++){
                    ++a;
                }
            }).start();
        }
    }
}
```

a的值肯定是小于10*1000的，这个就是因为原子性的问题。因为在java里面加法并不是原子操作，可能一个宪政在执行加法操作的时候，别的线程将CPU时间抢夺过去了，导致这个线程被挂起，所以线程将值放回主内存的时候会导致值覆盖的问题，所以才会小于10000的。

#### 禁止指令重排

在底层实现的时候，cpu 是会按照自己认为的最佳执行顺序来执行程序的。所以使用 volatile 来告诉操作系统，你别搞，让我自己决定我的执行顺序是什么。所以如果要完善单例模式，那么就要用这个`volatile`来修饰`instance`。

好了稍微的的扯了一下别的，扯得有些远了，回到AQS来

### 回到AQS

继续上面的`state`，这个`state`就是共享资源

```java
/**
* The synchronization state.
*/
private volatile int state;
```

而队列就是一个FIFO的双向队列，如果当前线程竞争失败了，那么AQS会把当前线程以及等待状态信息构造成一个Node(包含当前线程Thread的类)放进这个队列，并且阻塞这个线程，当获取到锁的线程释放之后，会唤醒队列中的堵塞的线程。

AQS有两种资源共享方式:

+ 独占
+ 共享

独占是只能有一个线程运行，例如`ReentrantLock`，共享是可以有多个线程同时执行，比如`CountdownLatch`，`Semaphore`、`ReadWriteLock`，`CyclicBarrier` 

## ReentrantLock源码分析

锁，那肯定得从`lock()`来分析。

```java
public void lock() {
        sync.acquire(1);
}
```

这个api就是获取资源锁的作用。如果它没有被另一个线程抢走并返回立即执行，`state`设置为1。如果有一个线程正在被执行，那么`state`就加1，因此我们可以知道`ReentrantLock`是可重入的。

```java
public final void acquire(int arg) {
    if (!tryAcquire(arg) &&
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
}
```

上面的这段代码位于`AbstractQueuedSynchronizer.java`中，这个方法的作用就是在独占模式下，至少执行一次`tryAcquire`并且返回`success`，如果线程在阻塞队列里面的时候，就一直执行`tryAcquire`，直到能够继续执行为止。那我们看下`tryAcquire`这个方法的实现吧。

```java
protected boolean tryAcquire(int arg) {
        throw new UnsupportedOperationException();
}
```

这个方法明显是要交给子类来实现的。所以看看子类的实现，一般创建一个`ReentrantLock`对象的时候，都是默认创建一个非公平锁的，所以看看非公平锁中是怎么实现的这个方法，哦对了提一句，非公平锁这个类名称是`NonFairLock`是`ReentrantLock`的静态内部类，也就是说下面的这段代码位于`ReentrantLock`中。

```java
final boolean nonfairTryAcquire(int acquires) {
    // 获取当前线程
    final Thread current = Thread.currentThread();
    // 获取锁计数器state的值
    int c = getState();
    if (c == 0) {
        // 如果是0的话，就使用CAS来设置state的值，
        if (compareAndSetState(0, acquires)) {
            // 设置独占线程的是当前线程
            setExclusiveOwnerThread(current);
            // 设置成功了返回true
            return true;
        }
    }
    // 如果state!=0但是当前的线程是独占的线程(抢到锁的线程)
    // 可重入的原理
    else if (current == getExclusiveOwnerThread()) {
        // 将锁计数器设置为当前有堵塞线程的个数
        int nextc = c + acquires;
        if (nextc < 0) // overflow
            throw new Error("Maximum lock count exceeded");
        setState(nextc);
        return true;
    }
    return false;
}
```


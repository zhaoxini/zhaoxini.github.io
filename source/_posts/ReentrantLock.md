---
title:[转载]深入理解ReentrantLock的实现原理
date: 2020-07-02
tags:
---



# [转载]深入理解ReentrantLock的实现原理



转：[深入理解ReentrantLock的实现原理](https://juejin.cn/post/6844903805683761165)



## ReentrantLock简介

`ReentrantLock`是`Java`在`JDK1.5`引入的显式锁，在实现原理和功能上都和内置锁(synchronized)上都有区别，在文章最后我们再比较这两个锁。
 首先我们要知道`ReentrantLock`是基于`AQS`实现的，所以我们得对`AQS`有所了解才能更好的去学习掌握`ReentrantLock`，关于`AQS`的介绍可以参考我之前写的一篇文章[《一文带你快速掌握AQS》](https://ddnd.cn/2019/03/15/java-abstractqueuedsynchronizer/)，这里简单回顾下`AQS`。



## ReentrantLock原理

通过前面的回顾，是不是对`ReentrantLock`有了一定的了解了，`ReentrantLock`通过重写**锁获取方式**和**锁释放方式**这两个方法实现了**公平锁**和**非公平锁**，那么`ReentrantLock`是怎么重写的呢，这也就是本节需要探讨的问题。

### ReentrantLock结构

![img](https://user-gold-cdn.xitu.io/2019/3/23/169aa1836d0ac5b0?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

首先`ReentrantLock`继承自父类`Lock`，然后有`3`个内部类，其中`Sync`内部类继承自`AQS`，另外的两个内部类继承自`Sync`，这两个类分别是用来**公平锁和非公平锁**的。
 通过`Sync`重写的方法`tryAcquire`、`tryRelease`可以知道，**`ReentrantLock`实现的是`AQS`的独占模式，也就是独占锁，这个锁是悲观锁**。

`ReentrantLock`有个重要的成员变量：

```
private final Sync sync;
复制代码
```

这个变量是用来指向`Sync`的子类的，也就是`FairSync`或者`NonfairSync`，这个也就是多态的**父类引用指向子类**，具体`Sycn`指向哪个子类，看构造方法：

```
public ReentrantLock() {
    sync = new NonfairSync();
}

public ReentrantLock(boolean fair) {
    sync = fair ? new FairSync() : new NonfairSync();
}
复制代码
```

`ReentrantLock`有两个构造方法，无参构造方法默认是创建**非公平锁**，而传入`true`为参数的构造方法创建的是**公平锁**。

### 非公平锁的实现原理

当我们使用无参构造方法构造的时候即`ReentrantLock lock = new ReentrantLock()`，创建的就是非公平锁。

```
public ReentrantLock() {
    sync = new NonfairSync();
}

//或者传入false参数 创建的也是非公平锁
public ReentrantLock(boolean fair) {
    sync = fair ? new FairSync() : new NonfairSync();
}
```

#### lock方法获取锁

1. `lock`方法调用`CAS`方法设置`state`的值，如果`state`等于期望值`0`(代表锁没有被占用)，那么就将`state`更新为`1`(代表该线程获取锁成功)，然后执行`setExclusiveOwnerThread`方法直接将该线程设置成锁的所有者。如果`CAS`设置`state`的值失败，即`state`不等于`0`，代表锁正在被占领着，则执行`acquire(1)`，即下面的步骤。
2. `nonfairTryAcquire`方法首先调用`getState`方法获取`state`的值，如果`state`的值为`0`(之前占领锁的线程刚好释放了锁)，那么用`CAS`这是`state`的值，设置成功则将该线程设置成锁的所有者，并且返回`true`。如果`state`的值不为`0`，那就**调用`getExclusiveOwnerThread`方法查看占用锁的线程是不是自己**，如果是的话那就直接将`state + 1`，然后返回`true`。如果`state`不为`0`且锁的所有者又不是自己，那就返回`false`，**然后线程会进入到同步队列中**。

![img](https://user-gold-cdn.xitu.io/2019/3/23/169aab7befb2e5de?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

```
final void lock() {
    //CAS操作设置state的值
    if (compareAndSetState(0, 1))
        //设置成功 直接将锁的所有者设置为当前线程 流程结束
        setExclusiveOwnerThread(Thread.currentThread());
    else
        //设置失败 则进行后续的加入同步队列准备
        acquire(1);
}

public final void acquire(int arg) {
    //调用子类重写的tryAcquire方法 如果tryAcquire方法返回false 那么线程就会进入同步队列
    if (!tryAcquire(arg) &&
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
}

//子类重写的tryAcquire方法
protected final boolean tryAcquire(int acquires) {
    //调用nonfairTryAcquire方法
    return nonfairTryAcquire(acquires);
}

final boolean nonfairTryAcquire(int acquires) {
    final Thread current = Thread.currentThread();
    int c = getState();
    //如果状态state=0，即在这段时间内 锁的所有者把锁释放了 那么这里state就为0
    if (c == 0) {
        //使用CAS操作设置state的值
        if (compareAndSetState(0, acquires)) {
            //操作成功 则将锁的所有者设置成当前线程 且返回true，也就是当前线程不会进入同步
            //队列。
            setExclusiveOwnerThread(current);
            return true;
        }
    }
    //如果状态state不等于0，也就是有线程正在占用锁，那么先检查一下这个线程是不是自己
    else if (current == getExclusiveOwnerThread()) {
        //如果线程就是自己了，那么直接将state+1，返回true，不需要再获取锁 因为锁就在自己
        //身上了。
        int nextc = c + acquires;
        if (nextc < 0) // overflow
            throw new Error("Maximum lock count exceeded");
        setState(nextc);
        return true;
    }
    //如果state不等于0，且锁的所有者又不是自己，那么线程就会进入到同步队列。
    return false;
}
复制代码
```

#### tryRelease锁的释放

1. 判断当前线程是不是锁的所有者，如果是则进行步骤`2`，如果不是则抛出异常。
2. 判断此次释放锁后`state`的值是否为0，如果是则代表**锁有没有重入**，然后将锁的所有者设置成`null`且返回`true`，然后执行步骤`3`，如果不是则**代表锁发生了重入**执行步骤`4`。
3. 现在锁已经释放完，即`state=0`，唤醒同步队列中的后继节点进行锁的获取。
4. 锁还没有释放完，即`state!=0`，不唤醒同步队列。

![img](https://user-gold-cdn.xitu.io/2019/3/23/169aad4a8e578933?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

```
public void unlock() {
    sync.release(1);
}

public final boolean release(int arg) {
    //子类重写的tryRelease方法，需要等锁的state=0，即tryRelease返回true的时候，才会去唤醒其
    //它线程进行尝试获取锁。
    if (tryRelease(arg)) {
        Node h = head;
        if (h != null && h.waitStatus != 0)
            unparkSuccessor(h);
        return true;
    }
    return false;
}
    
protected final boolean tryRelease(int releases) {
    //状态的state减去releases
    int c = getState() - releases;
    //判断锁的所有者是不是该线程
    if (Thread.currentThread() != getExclusiveOwnerThread())
        //如果所的所有者不是该线程 则抛出异常 也就是锁释放的前提是线程拥有这个锁，
        throw new IllegalMonitorStateException();
    boolean free = false;
    //如果该线程释放锁之后 状态state=0，即锁没有重入，那么直接将将锁的所有者设置成null
    //并且返回true，即代表可以唤醒其他线程去获取锁了。如果该线程释放锁之后state不等于0，
    //那么代表锁重入了，返回false，代表锁还未正在释放，不用去唤醒其他线程。
    if (c == 0) {
        free = true;
        setExclusiveOwnerThread(null);
    }
    setState(c);
    return free;
}
复制代码
```

### 公平锁的实现原理

#### lock方法获取锁

1. 获取状态的`state`的值，如果`state=0`即代表锁没有被其它线程占用(但是并不代表同步队列没有线程在等待)，执行步骤`2`。如果`state!=0`则代表锁正在被其它线程占用，执行步骤`3`。
2. **判断同步队列是否存在线程(节点)，如果不存在则直接将锁的所有者设置成当前线程，且更新状态state，然后返回true。**
3. **判断锁的所有者是不是当前线程，如果是则更新状态state的值，然后返回true，如果不是，那么返回false，即线程会被加入到同步队列中**

通过步骤`2`**实现了锁获取的公平性，即锁的获取按照先来先得的顺序，后来的不能抢先获取锁，非公平锁和公平锁也正是通过这个区别来实现了锁的公平性。**

![img](data:image/svg+xml;utf8,<?xml version="1.0"?><svg xmlns="http://www.w3.org/2000/svg" version="1.1" width="878" height="662"></svg>)

```
final void lock() {
    acquire(1);
}

public final void acquire(int arg) {
    //同步队列中有线程 且 锁的所有者不是当前线程那么将线程加入到同步队列的尾部，
    //保证了公平性，也就是先来的线程先获得锁，后来的不能抢先获取。
    if (!tryAcquire(arg) &&
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
}

protected final boolean tryAcquire(int acquires) {
    final Thread current = Thread.currentThread();
    int c = getState();
    //判断状态state是否等于0，等于0代表锁没有被占用，不等于0则代表锁被占用着。
    if (c == 0) {
        //调用hasQueuedPredecessors方法判断同步队列中是否有线程在等待，如果同步队列中没有
        //线程在等待 则当前线程成为锁的所有者，如果同步队列中有线程在等待，则继续往下执行
        //这个机制就是公平锁的机制，也就是先让先来的线程获取锁，后来的不能抢先获取。
        if (!hasQueuedPredecessors() &&
            compareAndSetState(0, acquires)) {
            setExclusiveOwnerThread(current);
            return true;
        }
    }
    //判断当前线程是否为锁的所有者，如果是，那么直接更新状态state，然后返回true。
    else if (current == getExclusiveOwnerThread()) {
        int nextc = c + acquires;
        if (nextc < 0)
            throw new Error("Maximum lock count exceeded");
        setState(nextc);
        return true;
    }
    //如果同步队列中有线程存在 且 锁的所有者不是当前线程，则返回false。
    return false;
}
复制代码
```

#### tryRelease锁的释放

公平锁的释放和非公平锁的释放一样，这里就不重复。
 公平锁和非公平锁的公平性是在**获取锁**的时候体现出来的，释放的时候都是一样释放的。

### lockInterruptibly可中断方式获取锁

`ReentrantLock`相对于`Synchronized`拥有一些更方便的特性，比如可以中断的方式去获取锁。

```
public void lockInterruptibly() throws InterruptedException {
    sync.acquireInterruptibly(1);
}

public final void acquireInterruptibly(int arg)
        throws InterruptedException {
    //如果当前线程已经中断了，那么抛出异常
    if (Thread.interrupted())
        throw new InterruptedException();
    //如果当前线程仍然未成功获取锁，则调用doAcquireInterruptibly方法，这个方法和
    //acquireQueued方法没什么区别，就是线程在等待状态的过程中，如果线程被中断，线程会
    //抛出异常。
    if (!tryAcquire(arg))
        doAcquireInterruptibly(arg);
}
复制代码
```

### tryLock超时等待方式获取锁

`ReentrantLock`除了能以能中断的方式去获取锁，还可以以超时等待的方式去获取锁，所谓超时等待就是线程如果在超时时间内没有获取到锁，那么就会返回`false`，而不是一直"死循环"获取。

1. 判断当前节点是否已经中断，已经被中断过则抛出异常，如果没有被中断过则尝试获取锁，获取失败则调用`doAcquireNanos`方法使用超时等待的方式获取锁。
2. 将当前节点封装成独占模式的节点加入到同步队列的队尾中。
3. 进入到"死循环"中，**但是这个死循环是有个限制的，也就是当线程达到超时时间了仍未获得锁，那么就会返回`false`，结束循环**。这里调用的是`LockSupport.parkNanos`方法，在超时时间内没有被中断，那么线程会从**超时等待状态转成了就绪状态**，然后被`CPU`调度继续执行循环，**而这时候线程已经达到超时等到的时间，返回false**。

> `LockSuport`的方法能响应`Thread.interrupt`，但是不会抛出异常

![img](data:image/svg+xml;utf8,<?xml version="1.0"?><svg xmlns="http://www.w3.org/2000/svg" version="1.1" width="931" height="877"></svg>)

```
public boolean tryLock(long timeout, TimeUnit unit)
        throws InterruptedException {
    return sync.tryAcquireNanos(1, unit.toNanos(timeout));
}

public final boolean tryAcquireNanos(int arg, long nanosTimeout)
        throws InterruptedException {
    //如果当前线程已经中断了  则抛出异常
    if (Thread.interrupted())
        throw new InterruptedException();
    //再尝试获取一次 如果不成功则调用doAcquireNanos方法进行超时等待获取锁
    return tryAcquire(arg) ||
        doAcquireNanos(arg, nanosTimeout);
}

private boolean doAcquireNanos(int arg, long nanosTimeout)
        throws InterruptedException {
    if (nanosTimeout <= 0L)
        return false;
    //计算超时的时间 即当前虚拟机的时间+设置的超时时间
    final long deadline = System.nanoTime() + nanosTimeout;
    //调用addWaiter将当前线程封装成独占模式的节点 并且加入到同步队列尾部
    final Node node = addWaiter(Node.EXCLUSIVE);
    boolean failed = true;
    try {
        for (;;) {
            final Node p = node.predecessor();
            //如果当前节点的前驱节点为头结点 则让当前节点去尝试获取锁。
            if (p == head && tryAcquire(arg)) {
                //当前节点获取锁成功 则将当前节点设置为头结点，然后返回true。
                setHead(node);
                p.next = null; // help GC
                failed = false;
                return true;
            }
            //如果当前节点的前驱节点不是头结点 或者 当前节点获取锁失败，
            //则再次判断当前线程是否已经超时。
            nanosTimeout = deadline - System.nanoTime();
            if (nanosTimeout <= 0L)
                return false;
            //调用shouldParkAfterFailedAcquire方法，告诉当前节点的前驱节点 我要进入
            //等待状态了，到我了记得喊我，即做好进入等待状态前的准备。
            if (shouldParkAfterFailedAcquire(p, node) &&
                nanosTimeout > spinForTimeoutThreshold)
                //调用LockSupport.parkNanos方法，将当前线程设置成超时等待的状态。
                LockSupport.parkNanos(this, nanosTimeout);
            if (Thread.interrupted())
                throw new InterruptedException();
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
复制代码
```

### ReentrantLock的等待/通知机制

我们知道关键字`Synchronized` + `Object`的`wait`和`notify`、`notifyAll`方法能实现**等待/通知**机制，那么`ReentrantLock`是否也能实现这样的等待/通知机制，答案是：可以。
 `ReentrantLock`通过`Condition`对象，也就是**条件队列**实现了和`wait`、`notify`、`notifyAll`相同的语义。 线程执行`condition.await()`方法，将节点1从同步队列转移到条件队列中。

![img](data:image/svg+xml;utf8,<?xml version="1.0"?><svg xmlns="http://www.w3.org/2000/svg" version="1.1" width="1210" height="261"></svg>)

线程执行`condition.signal()`方法，将节点1从条件队列中转移到同步队列。

![img](data:image/svg+xml;utf8,<?xml version="1.0"?><svg xmlns="http://www.w3.org/2000/svg" version="1.1" width="1217" height="223"></svg>)

因为只有在同步队列中的线程才能去获取锁，所以通过`Condition`对象的`wait`和`signal`方法能实现等待/通知机制。
 代码示例：

```
ReentrantLock lock = new ReentrantLock();
Condition condition = lock.newCondition();
public void await() {
    lock.lock();
    try {
        System.out.println("线程获取锁----" + Thread.currentThread().getName());
        condition.await(); //调用await()方法 会释放锁，和Object.wait()效果一样。
        System.out.println("线程被唤醒----" + Thread.currentThread().getName());
    } catch (InterruptedException e) {
        e.printStackTrace();
    } finally {
        lock.unlock();
        System.out.println("线程释放锁----" + Thread.currentThread().getName());
    }
}

public void signal() {
    try {
        Thread.sleep(1000);  //休眠1秒钟 等等一个线程先执行
    } catch (InterruptedException e) {
        e.printStackTrace();
    }
    lock.lock();
    try {
        System.out.println("另外一个线程获取到锁----" + Thread.currentThread().getName());
        condition.signal();
        System.out.println("唤醒线程----" + Thread.currentThread().getName());
    } finally {
        lock.unlock();
        System.out.println("另外一个线程释放锁----" + Thread.currentThread().getName());
    }
}

public static void main(String[] args) {
    Test t = new Test();
    Thread t1 = new Thread(new Runnable() {
        @Override
        public void run() {
            t.await();
        }
    });

    Thread t2 = new Thread(new Runnable() {
        @Override
        public void run() {
            t.signal();
        }
    });

    t1.start();
    t2.start();
}
复制代码
```

运行输出：

```
线程获取锁----Thread-0
另外一个线程获取到锁----Thread-1
唤醒线程----Thread-1
另外一个线程释放锁----Thread-1
线程被唤醒----Thread-0
线程释放锁----Thread-0
复制代码
```

执行的流程大概是这样，线程`t1`先获取到锁，输出了"线程获取锁----Thread-0"，然后线程`t1`调用`await`方法，调用这个方法的结果就是**线程`t1`释放了锁进入等待状态，等待唤醒**，接下来线程`t2`获取到锁，然输出了"另外一个线程获取到锁----Thread-1"，同时线程`t2`调用`signal`方法，调用这个方法的结果就是**唤醒一个在条件队列(Condition)的线程，然后线程`t1`被唤醒，而这个时候线程`t2`并没有释放锁，线程`t1`也就没法获得锁，等线程`t2`继续执行输出"唤醒线程----Thread-1"之后线程`t2`释放锁且输出"另外一个线程释放锁----Thread-1"，这时候线程`t1`获得锁，继续往下执行输出了`线程被唤醒----Thread-0`，然后释放锁输出"线程释放锁----Thread-0"**。

如果想单独唤醒部分线程应该怎么做呢？这时就有必要使用多个`Condition`对象了，因为`ReentrantLock`支持创建多个`Condition`对象，例如：

```
//为了减少篇幅 仅给出伪代码
ReentrantLock lock = new ReentrantLock();
Condition condition = lock.newCondition();
Condition condition1 = lock.newCondition();

//线程1 调用condition.await() 线程进入到条件队列
condition.await();

//线程2 调用condition1.await() 线程进入到条件队列
condition1.await();

//线程32 调用condition.signal() 仅唤醒调用condition中的线程，不会影响到调用condition1。
condition1.await();
复制代码
```

这样就实现了部分唤醒的功能。

## ReentrantLock和Synchronized对比

关于`Synchronized`的介绍可以看[《synchronized的使用（一）》](https://ddnd.cn/2019/03/21/java-synchronized/)、[《深入分析synchronized原理和锁膨胀过程(二)》](https://ddnd.cn/2019/03/22/java-synchronized-2/)

|                     | ReentrantLock  | Synchronized                                                 |
| ------------------- | -------------- | ------------------------------------------------------------ |
| 底层实现            | 通过`AQS`实现  | 通过`JVM`实现，其中`synchronized`又有多个类型的锁，除了重量级锁是通过`monitor`对象(操作系统mutex互斥原语)实现外，其它类型的通过对象头实现。 |
| 是否可重入          | 是             | 是                                                           |
| 公平锁              | 是             | 否                                                           |
| 非公平锁            | 是             | 是                                                           |
| 锁的类型            | 悲观锁、显式锁 | 悲观锁、隐式锁(内置锁)                                       |
| 是否支持中断        | 是             | 否                                                           |
| 是否支持超时等待    | 是             | 否                                                           |
| 是否自动获取/释放锁 | 否             | 是                                                           |

## 参考

《Java并发编程的艺术》
 [深入理解AbstractQueuedSynchronizer(AQS)](https://juejin.im/post/6844903601538596877#heading-10)
 [Java 重入锁 ReentrantLock 原理分析)](https://www.imooc.com/article/28934)



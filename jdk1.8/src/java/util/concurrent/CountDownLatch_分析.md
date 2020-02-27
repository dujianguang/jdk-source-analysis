# CountDownLatch

> 在并发编程的场景中，最常见的一个case是某个任务的执行，需要等到多个线程都执行完毕之后才可以进行，CountDownLatch可以很好解决这个问题

闭锁（Latch）：一种同步方法，可以延迟线程的进度直到线程到达某个终点状态。通俗的讲就是，一个闭锁相当于一扇大门，在大门打开之前所有线程都被阻断，一旦大门打开所有线程都将通过，但是一旦大门打开，所有线程都通过了，那么这个闭锁的状态就失效了，门的状态也就不能变了，只能是打开状态。也就是说闭锁的状态是一次性的，它确保在闭锁打开之前所有特定的活动都需要在闭锁打开之后才能完成。

**CountDownLatch**是JDK 5+里面闭锁的一个实现，允许一个或者多个线程等待某个事件的发生。**CountDownLatch**有一个正数计数器，*countDown*方法对计数器做减操作，*await*方法等待计数器达到0。所有*await*的线程都会阻塞直到计数器为0或者等待线程中断或者超时。

## 1、CountDownLatch是什么？

### 1.1、接口定义

```java
// 构造器，必须指定一个大于零的计数
public CountDownLatch(int count) {
    if (count < 0) throw new IllegalArgumentException("count < 0");
    this.sync = new Sync(count);
}

// 线程阻塞，直到计数为0的时候唤醒；可以响应线程中断退出阻塞
public void await() throws InterruptedException {
    sync.acquireSharedInterruptibly(1);
}

// 线程阻塞一段时间，如果计数依然不是0，则返回false；否则返回true
public boolean await(long timeout, TimeUnit unit)
    throws InterruptedException {
    return sync.tryAcquireSharedNanos(1, unit.toNanos(timeout));
}

// 计数-1
public void countDown() {
    sync.releaseShared(1);
}

// 获取计数
public long getCount() {
    return sync.getCount();
}
```

### 2.2、demo示例

```java
import java.util.concurrent.CountDownLatch;

/**
 * 线程1实现 10加到100
 * 线程2实现 100加到200
 * 线程3实现 线程1和线程2计算结果的和
 */
public class CountDownLatchTest {
    private CountDownLatch countDownLatch;

    private int start = 10;
    private int mid = 100;
    private int end = 200;

    private volatile int tmpRes1, tmpRes2;

    private int add(int start, int end) {
        int sum = 0;
        for (int i = start; i <= end; i++) {
            sum += i;
        }
        return sum;
    }


    private int sum(int a, int b) {
        return a + b;
    }

    public void calculate() {
        countDownLatch = new CountDownLatch(2);

        Thread thread1 = new Thread(() -> {
            try {
                // 确保线程3先与1，2执行，由于countDownLatch计数不为0而阻塞
                Thread.sleep(100);
                System.out.println(Thread.currentThread().getName() + " : 开始执行");
                tmpRes1 = add(start, mid);
                System.out.println(Thread.currentThread().getName() +
                        " : calculate ans: " + tmpRes1);
            } catch (InterruptedException e) {
                e.printStackTrace();
            } finally {
                countDownLatch.countDown();
            }
        }, "线程1");

        Thread thread2 = new Thread(() -> {
            try {
                // 确保线程3先与1，2执行，由于countDownLatch计数不为0而阻塞
                Thread.sleep(100);
                System.out.println(Thread.currentThread().getName() + " : 开始执行");
                tmpRes2 = add(mid + 1, end);
                System.out.println(Thread.currentThread().getName() +
                        " : calculate ans: " + tmpRes2);
            } catch (InterruptedException e) {
                e.printStackTrace();
            } finally {
                countDownLatch.countDown();
            }
        }, "线程2");


        Thread thread3 = new Thread(() -> {
            try {
                System.out.println(Thread.currentThread().getName() + " : 开始执行");
                countDownLatch.await();
                int ans = sum(tmpRes1, tmpRes2);
                System.out.println(Thread.currentThread().getName() +
                        " : calculate ans: " + ans);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }, "线程3");

        thread3.start();
        thread1.start();
        thread2.start();
    }


    public static void main(String[] args) throws InterruptedException {
        CountDownLatchTest demo = new CountDownLatchTest();
        demo.calculate();

        Thread.sleep(1000);
    }
}

输出：
线程3 : 开始执行
线程2 : 开始执行
线程1 : 开始执行
线程2 : calculate ans: 15050
线程1 : calculate ans: 5005
线程3 : calculate ans: 20055
```

看了上面的定义和Demo之后，使用就会简单一点了，一般流程如

1. 首先是创建实例 CountDownLatch countDown = new CountDownLatch(2)
2. 需要同步的线程执行完之后，计数-1； countDown.countDown()
3. 需要等待其他线程执行完毕之后，再运行的线程，调用 countDown.await()实现阻塞同步

**注意**

- 在创建实例是，必须指定初始的计数值，且应大于0
- 必须有线程中显示的调用了countDown()计数-1方法；必须有线程显示调用了 await()方法（没有这个就没有必要使用CountDownLatch了）
- 由于await()方法会阻塞到计数为0，如果在代码逻辑中某个线程漏掉了计数-1，导致最终计数一直大于0，直接导致死锁了
- 鉴于上面一点，更多的推荐 await(long, TimeUnit)来替代直接使用await()方法，至少不会造成阻塞死只能重启的情况

有兴趣的小伙伴可以对比下这个实现与 [《Java并发学习之ReentrantLock的工作原理及使用姿势》](https://my.oschina.net/u/566591/blog/1557978)中的demo，明显感觉使用CountDownLatch优雅得多（后面有机会介绍用更有意思的Fork/Join来实现累加）

### 1.3、应用场景

前面给了一个demo演示如何用，那这个东西在实际的业务场景中是否会用到呢？

因为确实在一个业务场景中使用到了，不然也就不会单独捞出这一节...

电商的详情页，由众多的数据拼装组成，如可以分成一下几个模块

- 交易的收发货地址，销量
- 商品的基本信息（标题，图文详情之类的）
- 推荐的商品列表
- 评价的内容
- ....

上面的几个模块信息，都是从不同的服务获取信息，且彼此没啥关联；所以为了提高响应，完全可以做成并发获取数据，如

- 线程1获取交易相关数据
- 线程2获取商品基本信息
- 线程3获取推荐的信息
- 线程4获取评价信息
- ....

但是最终拼装数据并返回给前端，需要等到上面的所有信息都获取完毕之后，才能返回，这个场景就非常的适合 CountDownLatch来做了

1. 在拼装完整数据的线程中调用 CountDownLatch#await(long, TimeUnit) 等待所有的模块信息返回
2. 每个模块信息的获取，由一个独立的线程执行；执行完毕之后调用 CountDownLatch#countDown() 进行计数-1

## 2、CountDownLatch原理

### 2.1、实现原理

同ReentrantLock一样，依然是借助AQS的双端队列，来实现原子的计数-1，线程阻塞和唤醒

### 2.2、AbstractQueuedSynchronizer

> AQS是一个用于构建锁和同步容器的框架。事实上concurrent包内许多类都是基于AQS构建，例如ReentrantLock，Semaphore，CountDownLatch，ReentrantReadWriteLock，FutureTask等。AQS解决了在实现同步容器时设计的大量细节问题

AQS使用一个FIFO的队列表示排队等待锁的线程，队列头节点称作“哨兵节点”或者“哑节点”，它不与任何线程关联。其他的节点与等待线程关联，每个节点维护一个等待状态waitStatus

```java
private transient volatile Node head;

private transient volatile Node tail;

private volatile int state;

static final class Node {
    static final Node SHARED = new Node();
    static final Node EXCLUSIVE = null;

    /** waitStatus value to indicate thread has cancelled */
    static final int CANCELLED =  1;
    /** waitStatus value to indicate successor's thread needs unparking */
    static final int SIGNAL    = -1;
    /** waitStatus value to indicate thread is waiting on condition */
    static final int CONDITION = -2;
    /**
     * waitStatus value to indicate the next acquireShared should
     * unconditionally propagate
     */
    static final int PROPAGATE = -3;

    //取值为 CANCELLED, SIGNAL, CONDITION, PROPAGATE 之一
    volatile int waitStatus;

    volatile Node prev;

    volatile Node next;

    // Link to next node waiting on condition, 
    // or the special value SHARED
    volatile Thread thread;

    Node nextWaiter;
}
```

### 2.3、计数器的初始化

CountDownLatch内部实现了AQS，并覆盖了tryAcquireShared()和tryReleaseShared()两个方法，下面说明干嘛用的。通过前面的使用，清楚了计数器的构造必须指定计数值,这个直接初始化了 AQS内部的state变量

```java
Sync(int count) {
	setState(count);
}
```

后续的计数-1/判断是否可用都是基于sate进行的

### 2.4、countDown() 计数-1

```java
// 计数-1
public void countDown() {
    sync.releaseShared(1);
}

public final boolean releaseShared(int arg) {
    if (tryReleaseShared(arg)) { // 首先尝试释放锁
        doReleaseShared();
        return true;
    }
    return false;
}

protected boolean tryReleaseShared(int releases) {
    // Decrement count; signal when transition to zero
    for (;;) {
        int c = getState();
        if (c == 0) //如果计数已经为0，则返回失败
            return false;
        int nextc = c-1;
        // 原子操作实现计数-1
        if (compareAndSetState(c, nextc)) 
            return nextc == 0;
    }
}

// 唤醒被阻塞的线程
private void doReleaseShared() {
    for (;;) {
        Node h = head;
        // 队列非空，表示有线程被阻塞
        if (h != null && h != tail) { 
            int ws = h.waitStatus;
            if (ws == Node.SIGNAL) { 
            // 头结点如果为SIGNAL,则唤醒头结点下个节点上关联的线程，并出队
                if (!compareAndSetWaitStatus(h, Node.SIGNAL, 0))
                    continue;            // loop to recheck cases
                unparkSuccessor(h);
            }
            else if (ws == 0 &&!compareAndSetWaitStatus(h, 0, Node.PROPAGATE))
                continue;                // loop on failed CAS
        }
        if (h == head) // 没有线程被阻塞，直接跳出
            break;
    }
}
```

上面截出计数减1的完整调用链

1. 尝试释放锁tryReleaseShared，实现计数-1

- 若计数已经小于0，则直接返回false
- 否则执行计数(AQS的state)减一
- 若减完之后，state==0，表示没有线程占用锁，即释放成功，然后就需要唤醒被阻塞的线程了

2. 释放并唤醒阻塞线程 doReleaseShared

- 如果队列为空，即表示没有线程被阻塞（也就是说没有线程调用了 CountDownLatch#wait()方法），直接退出
- 头结点如果为SIGNAL, 则依次唤醒头结点下个节点上关联的线程，并出队

**疑问一：** 看到这个实现，是不是只要countDownLatch的计数为0了，所有被阻塞的线程都会被执行？

改下上面的demo，新增线程4，实现线程2的结果-线程1的结果

```java
public class CountDownLatchDemo {
    
    // ...省略重复
    
    private int sub(int a, int b) {
        return a - b;
    }

    public void calculate() {
        countDownLatch = new CountDownLatch(2);

        Thread thread1 = // ... ;
        Thread thread2 = // ...;
        
        Thread thread3 = new Thread(()-> {
            try {
                System.out.println(Thread.currentThread().getName() + " : 开始执行");
                countDownLatch.await();
                System.out.println(Thread.currentThread().getName() + " : 唤醒");
                Thread.sleep(100); // 确保线程4先执行完相减
                int ans = sum(tmpRes1, tmpRes2);
                System.out.println(Thread.currentThread().getName() +
                        " : calculate ans: " + ans);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }, "线程3");

        Thread thread4 = new Thread(()-> {
            try {
                System.out.println(Thread.currentThread().getName() + " : 开始执行");
                countDownLatch.await();
                System.out.println(Thread.currentThread().getName() + " : 唤醒");
                int ans = sub(tmpRes2, tmpRes1);
                Thread.sleep(200); // 保证线程3先输出执行结果，以验证线程3和线程4是否并发执行
                System.out.println(Thread.currentThread().getName() +
                        " : calculate ans: " + ans);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }, "线程4");
        
        thread3.start();
        thread4.start();
        thread1.start();
        thread2.start();
    }

    public static void main(String[] args) throws InterruptedException {
        CountDownLatchDemo demo = new CountDownLatchDemo();
        demo.calculate();

        Thread.sleep(1000);
    }
}
输出：
线程4 : 开始执行
线程3 : 开始执行
线程2 : 开始执行
线程2 : calculate ans: 15050
线程1 : 开始执行
线程1 : calculate ans: 5005
线程3 : 唤醒
线程4 : 唤醒
线程3 : calculate ans: 20055
线程4 : calculate ans: 10045
```

上面的实现中，线程3中sleep一段时间，确保线程4的计算会优先执行；线程4计算完成之后的sleep时间，以保证线程3计算完成并输出结果，然后线程4才输出结果；结合输出，这个期望是准确的，也就是说，线程3和线程4被唤醒后是并发执行的，没有先后阻塞顺序

即CountDownLatch计数为0之后，所有被阻塞的线程都会被唤醒，且彼此相对独立，不会出现独占锁阻塞的问题

### 2.5、await() 阻塞

使当前线程在锁存器倒计数至零之前一直等待，除非线程被中断，或者等待超时

```java
public void await() throws InterruptedException {
    sync.acquireSharedInterruptibly(1);
}
    

public final void acquireSharedInterruptibly(int arg)
        throws InterruptedException {
    if (Thread.interrupted()) // 若线程中端，直接抛异常
        throw new InterruptedException();
    if (tryAcquireShared(arg) < 0)
        doAcquireSharedInterruptibly(arg);
}


// 计数为0时，表示获取锁成功
protected int tryAcquireShared(int acquires) {
    return (getState() == 0) ? 1 : -1;
}

// 阻塞，并入队
private void doAcquireSharedInterruptibly(int arg)
    throws InterruptedException {
    final Node node = addWaiter(Node.SHARED); // 入队
    boolean failed = true;
    try {
        for (;;) {
            // 获取前驱节点
            final Node p = node.predecessor();
            if (p == head) {
                int r = tryAcquireShared(arg);
                if (r >= 0) {
                    // 获取锁成功，设置队列头为node节点
                    setHeadAndPropagate(node, r);
                    p.next = null; // help GC
                    failed = false;
                    return;
                }
            }
            if (shouldParkAfterFailedAcquire(p, node) // 线程挂起
              && parkAndCheckInterrupt())
                throw new InterruptedException();
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
```

阻塞的逻辑相对简单

1. 判断state计数是否为0，不是，则直接放过执行后面的代码
2. 大于0，则表示需要阻塞等待计数为0
3. 当前线程封装Node对象，进入阻塞队列
4. 然后就是循环尝试获取锁，直到成功（即state为0）后出队，继续执行线程后续代码

## 3、小结

### 3.1、使用注意

- 在创建实例时，必须指定初始的计数值，且应大于0
- 必须有线程中显示的调用了countDown()计数-1方法；必须有线程显示调用了await()方法（没有这个就没有必要使用CountDownLatch了）
- 由于await()方法会阻塞到计数为0，如果在代码逻辑中某个线程漏掉了计数-1，导致最终计数一直大于0，直接导致死锁了；
- 鉴于上面一点，更多的推荐 await(long, TimeUnit)来替代直接使用await()方法，至少不会造成阻塞死只能重启的情况
- 允许多个线程调用await方法，当计数为0后，所有被阻塞的线程都会被唤醒

### 3.2、实现原理

**await内部实现流程:**

1. 判断state计数是否为0，不是，则直接放过执行后面的代码
2. 大于0，则表示需要阻塞等待计数为0
3. 当前线程封装Node对象，进入阻塞队列
4. 然后就是循环尝试获取锁，直到成功（即state为0）后出队，继续执行线程后续代码

**countDown内部实现流程:**

1. 尝试释放锁tryReleaseShared，实现计数-1

- 若计数已经小于0，则直接返回false
- 否则执行计数(AQS的state)减一
- 若减完之后，state==0，表示没有线程占用锁，即释放成功，然后就需要唤醒被阻塞的线程了

1. 释放并唤醒阻塞线程 doReleaseShared

- 如果队列为空，即表示没有线程被阻塞（也就是说没有线程调用了 CountDownLatch#wait()方法），直接退出
- 头结点如果为SIGNAL, 则依次唤醒头结点下个节点上关联的线程，并出队
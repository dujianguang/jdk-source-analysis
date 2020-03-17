

# ReentrantLock

## 1、介绍

ReentrantLock是一个可重入的互斥锁，又被称为“独占锁”。

顾名思义，ReentrantLock锁在同一个时间点只能被一个线程锁持有；而可重入的意思是，ReentrantLock锁，可以被单个线程多次获取。
ReentrantLock分为“**公平锁**”和“**非公平锁**”。它们的区别体现在获取锁的机制上是否公平。“锁”是为了保护竞争资源，防止多个线程同时操作线程而出错，ReentrantLock在同一个时间点只能被一个线程获取(当某线程获取到“锁”时，其它线程就必须等待)；ReentraantLock是通过一个FIFO的等待队列来管理获取该锁所有线程的。在“公平锁”的机制下，线程依次排队获取锁；而“非公平锁”在锁是可获取状态时，不管自己是不是在队列的开头都会获取锁。

ReentrantLock，可重入锁，是一种递归无阻塞的同步机制。它可以等同于 synchronized 的使用，但是 ReentrantLock 提供了比 synchronized 更强大、灵活的锁机制，可以减少死锁发生的概率。

> 一个可重入的互斥锁定 Lock，它具有与使用 synchronized 方法和语句所访问的隐式监视器锁定相同的一些基本行为和语义，但功能更强大。
>
> ReentrantLock 将由最近成功获得锁定，并且还没有释放该锁定的线程所拥有。当锁定没有被另一个线程所拥有时，调用 lock 的线程将成功获取该锁定并返回。如果当前线程已经拥有该锁定，此方法将立即返回。可以使用 #isHeldByCurrentThread() 和 #getHoldCount() 方法来检查此情况是否发生。

ReentrantLock 还提供了**公平锁**和**非公平锁**的选择，通过构造方法接受一个可选的 fair 参数（默认非公平锁）：当设置为 true 时，表示公平锁；否则为非公平锁。

公平锁与非公平锁的区别在于，公平锁的锁获取是有顺序的。但是公平锁的效率往往没有非公平锁的效率高，在许多线程访问的情况下，公平锁表现出较低的吞吐量。

### 1.1、数据结构

![reentrantlock_uml](https://github.com/muyutingfeng/jdk-source-analysis/raw/master/note/doc/java/util/concurrent/locks/reentrantlock_uml.jpg?raw=true)

从图中可以看出：

(01) ReentrantLock实现了Lock接口。

(02) ReentrantLock与sync是组合关系。ReentrantLock中，包含了Sync对象；而且，Sync是AQS的子类；更重要的是，Sync有两个子类FairSync(公平锁)和NonFairSync(非公平锁)。ReentrantLock是一个独占锁，至于它到底是公平锁还是非公平锁，就取决于sync对象是"FairSync的实例"还是"NonFairSync的实例"。



#### 1.1.1 Sync抽象类

Sync 是 ReentrantLock 的内部静态类，实现 AbstractQueuedSynchronizer 抽象类，同步器抽象类。它使用 AQS 的 state 字段，来表示当前锁的持有数量，从而实现**可重入**的特性。

##### 1.1.1.1 lock

```java
				/**
         * Performs {@link Lock#lock}. The main reason for subclassing
         * is to allow fast path for nonfair version.
         * 执行锁。抽象了该方法的原因是，允许子类实现快速获得非公平锁的逻辑。
         */
        abstract void lock();
```

##### 1.1.1.2 nonfairTryAcquire

```java
				/**
         * Performs non-fair tryLock.  tryAcquire is implemented in
         * subclasses, but both need nonfair try for trylock method.
         */
        final boolean nonfairTryAcquire(int acquires) {
            //当前线程
            final Thread current = Thread.currentThread();
            //获取同步状态
            int c = getState();
            //state == 0,表示没有该锁处于空闲状态
            if (c == 0) {
                //获取锁成功，设置为当前线程所有
                if (compareAndSetState(0, acquires)) {
                    setExclusiveOwnerThread(current);
                    return true;
                }
            }
            //线程重入
            //判断锁持有的线程是否为当前线程
            else if (current == getExclusiveOwnerThread()) {
                int nextc = c + acquires;
                if (nextc < 0) // overflow
                    throw new Error("Maximum lock count exceeded");
                setState(nextc);
                return true;
            }
            return false;
        }
```

该方法主要逻辑：首先判断同步状态 state == 0 ?

如果**是**，表示该锁还没有被线程持有，直接通过CAS获取同步状态。

如果成功，返回 true 。

否则，返回 false 。

如果**不是**，则判断当前线程是否为**获取锁的线程**？

如果是，则获取锁，成功返回 true 。成功获取锁的线程，再次获取锁，这是增加了同步状态 state 。通过这里的实现，我们可以看到上面提到的 “它使用 AQS 的 state 字段，来表示当前锁的持有数量，从而实现可重入的特性”。

否则，返回 false 。

理论来说，这个方法应该在子类 **FairSync** 中实现，但是为什么会在这里呢？在下文的 ReentrantLock.tryLock() 中，详细解析。

1.1.2.3 tryRelease

```java
protected final boolean tryRelease(int releases) {
            // 减掉releases
            int c = getState() - releases;
            // 如果释放的不是持有锁的线程，抛出异常
            if (Thread.currentThread() != getExclusiveOwnerThread())
                throw new IllegalMonitorStateException();
            boolean free = false;
            // state == 0 表示已经释放完全了，其他线程可以获取同步状态了
            if (c == 0) {
                free = true;
                setExclusiveOwnerThread(null);
            }
            setState(c);
            return free;
        }
```

通过判断判断是否为获得到锁的线程，保证该方法线程安全。

只有当同步状态彻底释放后，该方法才会返回 true 。当 state == 0 时，则将锁持有线程设置为 null ，free= true，表示释放成功。



### 1.2、函数列表

```java
// 创建一个 ReentrantLock ，默认是“非公平锁”。
ReentrantLock()
// 创建策略是fair的 ReentrantLock。fair为true表示是公平锁，fair为false表示是非公平锁。
ReentrantLock(boolean fair)
// 查询当前线程保持此锁的次数。
int getHoldCount()
// 返回目前拥有此锁的线程，如果此锁不被任何线程拥有，则返回 null。
protected Thread getOwner()
// 返回一个 collection，它包含可能正等待获取此锁的线程。
protected Collection<Thread> getQueuedThreads()
// 返回正等待获取此锁的线程估计数。
int getQueueLength()
// 返回一个 collection，它包含可能正在等待与此锁相关给定条件的那些线程。
protected Collection<Thread> getWaitingThreads(Condition condition)
// 返回等待与此锁相关的给定条件的线程估计数。
int getWaitQueueLength(Condition condition)
// 查询给定线程是否正在等待获取此锁。
boolean hasQueuedThread(Thread thread)
// 查询是否有些线程正在等待获取此锁。
boolean hasQueuedThreads()
// 查询是否有些线程正在等待与此锁有关的给定条件。
boolean hasWaiters(Condition condition)
// 如果是“公平锁”返回true，否则返回false。
boolean isFair()
// 查询当前线程是否保持此锁。
boolean isHeldByCurrentThread()
// 查询此锁是否由任意线程保持。
boolean isLocked()
// 获取锁。
void lock()
// 如果当前线程未被中断，则获取锁。
void lockInterruptibly()
// 返回用来与此 Lock 实例一起使用的 Condition 实例。
Condition newCondition()
// 仅在调用时锁未被另一个线程保持的情况下，才获取该锁。
boolean tryLock()
// 如果锁在给定等待时间内没有被另一个线程保持，且当前线程未被中断，则获取该锁。
boolean tryLock(long timeout, TimeUnit unit)
// 试图释放此锁。
void unlock()
```



### 1.3、示例

```java
import java.util.concurrent.locks.Lock;
import java.util.concurrent.locks.ReentrantLock;

// LockTest1.java
// 仓库
class Depot { 
    private int size;        // 仓库的实际数量
    private Lock lock;        // 独占锁

    public Depot() {
        this.size = 0;
        this.lock = new ReentrantLock();
    }

    public void produce(int val) {
        lock.lock();
        try {
            size += val;
            System.out.printf("%s produce(%d) --> size=%d\n", 
                    Thread.currentThread().getName(), val, size);
        } finally {
            lock.unlock();
        }
    }

    public void consume(int val) {
        lock.lock();
        try {
            size -= val;
            System.out.printf("%s consume(%d) <-- size=%d\n", 
                    Thread.currentThread().getName(), val, size);
        } finally {
            lock.unlock();
        }
    }
}; 

// 生产者
class Producer {
    private Depot depot;
    
    public Producer(Depot depot) {
        this.depot = depot;
    }

    // 消费产品：新建一个线程向仓库中生产产品。
    public void produce(final int val) {
        new Thread() {
            public void run() {
                depot.produce(val);
            }
        }.start();
    }
}

// 消费者
class Customer {
    private Depot depot;
    
    public Customer(Depot depot) {
        this.depot = depot;
    }

    // 消费产品：新建一个线程从仓库中消费产品。
    public void consume(final int val) {
        new Thread() {
            public void run() {
                depot.consume(val);
            }
        }.start();
    }
}

public class LockTest1 {  
    public static void main(String[] args) {  
        Depot mDepot = new Depot();
        Producer mPro = new Producer(mDepot);
        Customer mCus = new Customer(mDepot);

        mPro.produce(60);
        mPro.produce(120);
        mCus.consume(90);
        mCus.consume(150);
        mPro.produce(110);
    }
}

输出：
Thread-0 produce(60) --> size=60
Thread-1 produce(120) --> size=180
Thread-3 consume(150) <-- size=30
Thread-2 consume(90) <-- size=-60
Thread-4 produce(110) --> size=50
```

**结果分析**：
(01) Depot 是个仓库。通过produce()能往仓库中生产货物，通过consume()能消费仓库中的货物。通过独占锁lock实现对仓库的互斥访问：在操作(生产/消费)仓库中货品前，会先通过lock()锁住仓库，操作完之后再通过unlock()解锁。
(02) Producer是生产者类。调用Producer中的produce()函数可以新建一个线程往仓库中生产产品。
(03) Customer是消费者类。调用Customer中的consume()函数可以新建一个线程消费仓库中的产品。
(04) 在主线程main中，我们会新建1个生产者mPro，同时新建1个消费者mCus。它们分别向仓库中生产/消费产品。
根据main中的生产/消费数量，仓库最终剩余的产品应该是50。运行结果是符合我们预期的！

这个模型存在两个问题：
(01) 现实中，仓库的容量不可能为负数。但是，此模型中的仓库容量可以为负数，这与现实相矛盾！
(02) 现实中，仓库的容量是有限制的。但是，此模型中的容量确实没有限制的！
这两个问题，我们稍微会讲到如何解决。现在，先看个简单的示例2；通过对比“示例1”和“示例2”,我们能更清晰的认识lock(),unlock()的用途。

示例2

```java
import java.util.concurrent.locks.Lock;
import java.util.concurrent.locks.ReentrantLock;

// LockTest2.java
// 仓库
class Depot { 
    private int size;        // 仓库的实际数量
    private Lock lock;        // 独占锁

    public Depot() {
        this.size = 0;
        this.lock = new ReentrantLock();
    }

    public void produce(int val) {
//        lock.lock();
//        try {
            size += val;
            System.out.printf("%s produce(%d) --> size=%d\n", 
                    Thread.currentThread().getName(), val, size);
//        } catch (InterruptedException e) {
//        } finally {
//            lock.unlock();
//        }
    }

    public void consume(int val) {
//        lock.lock();
//        try {
            size -= val;
            System.out.printf("%s consume(%d) <-- size=%d\n", 
                    Thread.currentThread().getName(), val, size);
//        } finally {
//            lock.unlock();
//        }
    }
}; 

// 生产者
class Producer {
    private Depot depot;
    
    public Producer(Depot depot) {
        this.depot = depot;
    }

    // 消费产品：新建一个线程向仓库中生产产品。
    public void produce(final int val) {
        new Thread() {
            public void run() {
                depot.produce(val);
            }
        }.start();
    }
}

// 消费者
class Customer {
    private Depot depot;
    
    public Customer(Depot depot) {
        this.depot = depot;
    }

    // 消费产品：新建一个线程从仓库中消费产品。
    public void consume(final int val) {
        new Thread() {
            public void run() {
                depot.consume(val);
            }
        }.start();
    }
}

public class LockTest2 {  
    public static void main(String[] args) {  
        Depot mDepot = new Depot();
        Producer mPro = new Producer(mDepot);
        Customer mCus = new Customer(mDepot);

        mPro.produce(60);
        mPro.produce(120);
        mCus.consume(90);
        mCus.consume(150);
        mPro.produce(110);
    }
}

输出：
Thread-0 produce(60) --> size=-60
Thread-4 produce(110) --> size=50
Thread-2 consume(90) <-- size=-60
Thread-1 produce(120) --> size=-60
Thread-3 consume(150) <-- size=-60
```

**结果说明**：
“示例2”在“示例1”的基础上去掉了lock锁。在“示例2”中，仓库中最终剩余的产品是-60，而不是我们期望的50。原因是我们没有实现对仓库的互斥访问。

 

**示例3**

在“示例3”中，我们通过Condition去解决“示例1”中的两个问题：“仓库的容量不可能为负数”以及“仓库的容量是有限制的”。
解决该问题是通过Condition。Condition是需要和Lock联合使用的：通过Condition中的await()方法，能让线程阻塞[类似于wait()]；通过Condition的signal()方法，能让唤醒线程[类似于notify()]。

```java
import java.util.concurrent.locks.Lock;
import java.util.concurrent.locks.ReentrantLock;
import java.util.concurrent.locks.Condition;

// LockTest3.java
// 仓库
class Depot {
    private int capacity;    // 仓库的容量
    private int size;        // 仓库的实际数量
    private Lock lock;        // 独占锁
    private Condition fullCondtion;            // 生产条件
    private Condition emptyCondtion;        // 消费条件

    public Depot(int capacity) {
        this.capacity = capacity;
        this.size = 0;
        this.lock = new ReentrantLock();
        this.fullCondtion = lock.newCondition();
        this.emptyCondtion = lock.newCondition();
    }

    public void produce(int val) {
        lock.lock();
        try {
             // left 表示“想要生产的数量”(有可能生产量太多，需多此生产)
            int left = val;
            while (left > 0) {
                // 库存已满时，等待“消费者”消费产品。
                while (size >= capacity)
                    fullCondtion.await();
                // 获取“实际生产的数量”(即库存中新增的数量)
                // 如果“库存”+“想要生产的数量”>“总的容量”，则“实际增量”=“总的容量”-“当前容量”。(此时填满仓库)
                // 否则“实际增量”=“想要生产的数量”
                int inc = (size+left)>capacity ? (capacity-size) : left;
                size += inc;
                left -= inc;
                System.out.printf("%s produce(%3d) --> left=%3d, inc=%3d, size=%3d\n", 
                        Thread.currentThread().getName(), val, left, inc, size);
                // 通知“消费者”可以消费了。
                emptyCondtion.signal();
            }
        } catch (InterruptedException e) {
        } finally {
            lock.unlock();
        }
    }

    public void consume(int val) {
        lock.lock();
        try {
            // left 表示“客户要消费数量”(有可能消费量太大，库存不够，需多此消费)
            int left = val;
            while (left > 0) {
                // 库存为0时，等待“生产者”生产产品。
                while (size <= 0)
                    emptyCondtion.await();
                // 获取“实际消费的数量”(即库存中实际减少的数量)
                // 如果“库存”<“客户要消费的数量”，则“实际消费量”=“库存”；
                // 否则，“实际消费量”=“客户要消费的数量”。
                int dec = (size<left) ? size : left;
                size -= dec;
                left -= dec;
                System.out.printf("%s consume(%3d) <-- left=%3d, dec=%3d, size=%3d\n", 
                        Thread.currentThread().getName(), val, left, dec, size);
                fullCondtion.signal();
            }
        } catch (InterruptedException e) {
        } finally {
            lock.unlock();
        }
    }

    public String toString() {
        return "capacity:"+capacity+", actual size:"+size;
    }
}; 

// 生产者
class Producer {
    private Depot depot;
    
    public Producer(Depot depot) {
        this.depot = depot;
    }

    // 消费产品：新建一个线程向仓库中生产产品。
    public void produce(final int val) {
        new Thread() {
            public void run() {
                depot.produce(val);
            }
        }.start();
    }
}

// 消费者
class Customer {
    private Depot depot;
    
    public Customer(Depot depot) {
        this.depot = depot;
    }

    // 消费产品：新建一个线程从仓库中消费产品。
    public void consume(final int val) {
        new Thread() {
            public void run() {
                depot.consume(val);
            }
        }.start();
    }
}

public class LockTest3 {  
    public static void main(String[] args) {  
        Depot mDepot = new Depot(100);
        Producer mPro = new Producer(mDepot);
        Customer mCus = new Customer(mDepot);

        mPro.produce(60);
        mPro.produce(120);
        mCus.consume(90);
        mCus.consume(150);
        mPro.produce(110);
    }
}	

输出：
Thread-0 produce( 60) --> left=  0, inc= 60, size= 60
Thread-1 produce(120) --> left= 80, inc= 40, size=100
Thread-2 consume( 90) <-- left=  0, dec= 90, size= 10
Thread-3 consume(150) <-- left=140, dec= 10, size=  0
Thread-4 produce(110) --> left= 10, inc=100, size=100
Thread-3 consume(150) <-- left= 40, dec=100, size=  0
Thread-4 produce(110) --> left=  0, inc= 10, size= 10
Thread-3 consume(150) <-- left= 30, dec= 10, size=  0
Thread-1 produce(120) --> left=  0, inc= 80, size= 80
Thread-3 consume(150) <-- left=  0, dec= 30, size= 50
```





## 2、获取公平锁

### lock

```java
/**
         * “当前线程”实际上是通过acquire(1)获取锁的。
         * 这里说明一下“1”的含义，它是设置“锁的状态”的参数。对于“独占锁”而言，锁处于可获取状态时，它的状态值是0；
         * 锁被线程初次获取到了，它的状态值就变成了1。
         *
         * 由于ReentrantLock(公平锁/非公平锁)是可重入锁，所以“独占锁”可以被单个线程多此获取，每获取1次就将锁的状态+1。
         * 也就是说，初次获取锁时，通过acquire(1)将锁的状态值设为1；再次获取锁时，将锁的状态值设为2；依次类推...这就是为什么获取锁时，
         * 传入的参数是1的原因了。可重入就是指锁可以被单个线程多次获取。
         */
        final void lock() {
            acquire(1);
        }
```

### acquire

(01) “当前线程”首先通过tryAcquire()尝试获取锁。获取成功的话，直接返回；尝试失败的话，进入到等待队列排序等待(前面还有可能有需要线程在等待该锁)。

(02) “当前线程”尝试失败的情况下，先通过addWaiter(Node.EXCLUSIVE)来将“当前线程”加入到"CLH队列(非阻塞的FIFO队列)"末尾。CLH队列就是线程等待队列。

(03) 在执行完addWaiter(Node.EXCLUSIVE)之后，会调用acquireQueued()来获取锁。由于此时ReentrantLock是公平锁，它会**根据公平性原则**来获取锁。

(04) “当前线程”在执行acquireQueued()时，会进入到CLH队列中休眠等待，直到获取锁了才返回！如果“当前线程”在休眠等待过程中被中断过，acquireQueued会返回true，此时"当前线程"会调用selfInterrupt()来自己给自己产生一个中断。

```java
/**
     * Acquires in exclusive mode, ignoring interrupts.  Implemented
     * by invoking at least once {@link #tryAcquire},
     * returning on success.  Otherwise the thread is queued, possibly
     * repeatedly blocking and unblocking, invoking {@link
     * #tryAcquire} until success.  This method can be used
     * to implement method {@link Lock#lock}.
     *
     * @param arg the acquire argument.  This value is conveyed to
     *        {@link #tryAcquire} but is otherwise uninterpreted and
     *        can represent anything you like.
     *
     *
     * 独占式获取同步状态
     *
     * 1、调用tryAcquire尝试获取锁，注意：tryAcquire在AbstractQueuedSynchronizer类中实现直接抛出异常，一般是子类NonfairSync、FairSync继承重写该方法
     * 2、tryAcquire会做如下尝试：
     *      a.如果state=0表示当前锁又没有被线程所持有，重新获取一次锁，成功返回true，失败返回false
     *      b.如果持有锁的线程就是当前线程，则将state累加1，用于记录重入次数，释放的时候也要全部释放，并返回true表示获取锁成功
     *      c.非以上两种情况，直接返回fasle
     * 3、如果获取锁成功，即tryAcquire返回true，则直接返回
     * 4、如果获取锁失败，即tryAcquire返回false，则将当前线程封装成Node放入到Sync Queue里(调用addWaiter)，等待Signal信号
     * 5、调用acquireQueued进行自旋的方式获取锁(有可能会 repeatedly blocking and unblocking)
     * 4、根据acquireQueued的返回值判断在获取lock的过程中是否被中断, 若被中断, 则自己再中断一下(selfInterrupt)
     *
     *  ------------------------------------------------------------------
     * #acquire(int arg) 方法，为 AQS 提供的模板方法。该方法为独占式获取同步状态，
     * 但是该方法对中断不敏感。也就是说，由于线程获取同步状态失败而加入到 CLH 同步
     * 队列中，后续对该线程进行中断操作时，线程不会从 CLH 同步队列中移除。
     *
     */
    public final void acquire(int arg) {
        //acquireQueued的主要作用是把已经追加到队列的线程节点（addWaiter方法返回值）进行阻塞，但阻塞前又通过tryAccquire重试是否能获得锁，如果重试成功能则无需阻塞，直接返回
        /**
         * 调用 #tryAcquire(int arg) 方法，去尝试获取同步状态，获取成功则设置锁状态并返回 true ，否则获取失败，返回 false 。
         * 若获取成功，#acquire(int arg) 方法，直接返回，不用线程阻塞，自旋直到获得同步状态成功
         */
        if (!tryAcquire(arg) &&
            acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
            selfInterrupt();
    }
```

### 1、tryAcquire

tryAcquire()的作用就是让“当前线程”尝试获取锁。获取成功返回true，失败则返回false。

#### tryAcquire

```java
/**
         * Fair version of tryAcquire.  Don't grant access unless
         * recursive call or no waiters or is first.
         *
         * tryAcquire()的作用就是尝试去获取锁。注意，这里只是尝试！
         * 尝试成功的话，返回true；尝试失败的话，返回false，后续再通过其它办法来获取该锁。
         * 后面我们会说明，在尝试失败的情况下，是如何一步步获取锁的。
         */
        protected final boolean tryAcquire(int acquires) {
            // 获取“当前线程”
            final Thread current = Thread.currentThread();
            // 获取“独占锁”的状态
            int c = getState();
            // c=0意味着“锁没有被任何线程锁拥有”，
            if (c == 0) {
                // 若“锁没有被任何线程锁拥有”，
                // 则判断“当前线程”是不是CLH队列中的第一个线程线程，
                // 若是的话，则获取该锁，设置锁的状态，并切设置锁的拥有者为“当前线程”。
                if (!hasQueuedPredecessors() &&
                    compareAndSetState(0, acquires)) {
                    setExclusiveOwnerThread(current);
                    return true;
                }
            }
            else if (current == getExclusiveOwnerThread()) {
                // 如果“独占锁”的拥有者已经为“当前线程”，
                // 则将更新锁的状态。
                int nextc = c + acquires;
                if (nextc < 0)
                    throw new Error("Maximum lock count exceeded");
                setState(nextc);
                return true;
            }
            return false;
        }
    }
```



#### hasQueuedPredecessors

```java
/**
     * Queries whether any threads have been waiting to acquire longer
     * than the current thread.
     *
     * <p>An invocation of this method is equivalent to (but may be
     * more efficient than):
     *  <pre> {@code
     * getFirstQueuedThread() != Thread.currentThread() &&
     * hasQueuedThreads()}</pre>
     *
     * <p>Note that because cancellations due to interrupts and
     * timeouts may occur at any time, a {@code true} return does not
     * guarantee that some other thread will acquire before the current
     * thread.  Likewise, it is possible for another thread to win a
     * race to enqueue after this method has returned {@code false},
     * due to the queue being empty.
     *
     * <p>This method is designed to be used by a fair synchronizer to
     * avoid <a href="AbstractQueuedSynchronizer#barging">barging</a>.
     * Such a synchronizer's {@link #tryAcquire} method should return
     * {@code false}, and its {@link #tryAcquireShared} method should
     * return a negative value, if this method returns {@code true}
     * (unless this is a reentrant acquire).  For example, the {@code
     * tryAcquire} method for a fair, reentrant, exclusive mode
     * synchronizer might look like this:
     *
     *  <pre> {@code
     * protected boolean tryAcquire(int arg) {
     *   if (isHeldExclusively()) {
     *     // A reentrant acquire; increment hold count
     *     return true;
     *   } else if (hasQueuedPredecessors()) {
     *     return false;
     *   } else {
     *     // try to acquire normally
     *   }
     * }}</pre>
     *
     * @return {@code true} if there is a queued thread preceding the
     *         current thread, and {@code false} if the current thread
     *         is at the head of the queue or the queue is empty
     * @since 1.7
     *
     * 通过判断"当前线程"是不是在CLH队列的队首，来返回AQS中是不是有比“当前线程”等待更久的线程。
     */
    public final boolean hasQueuedPredecessors() {
        // The correctness of this depends on head being initialized
        // before tail and on head.next being accurate if the current
        // thread is first in queue.
        Node t = tail; // Read fields in reverse initialization order
        Node h = head;
        Node s;
        return h != t &&
            ((s = h.next) == null || s.thread != Thread.currentThread());
    }
```



##### Node

Node就是CLH队列的节点。Node在AQS中实现，它的数据结构如下：

```java
private transient volatile Node head;    // CLH队列的队首
private transient volatile Node tail;    // CLH队列的队尾

// CLH队列的节点
static final class Node {
    static final Node SHARED = new Node();
    static final Node EXCLUSIVE = null;

    // 线程已被取消，对应的waitStatus的值
    static final int CANCELLED =  1;
    // “当前线程的后继线程需要被unpark(唤醒)”，对应的waitStatus的值。
    // 一般发生情况是：当前线程的后继线程处于阻塞状态，而当前线程被release或cancel掉，因此需要唤醒当前线程的后继线程。
    static final int SIGNAL    = -1;
    // 线程(处在Condition休眠状态)在等待Condition唤醒，对应的waitStatus的值
    static final int CONDITION = -2;
    // (共享锁)其它线程获取到“共享锁”，对应的waitStatus的值
    static final int PROPAGATE = -3;

    // waitStatus为“CANCELLED, SIGNAL, CONDITION, PROPAGATE”时分别表示不同状态，
    // 若waitStatus=0，则意味着当前线程不属于上面的任何一种状态。
    volatile int waitStatus;

    // 前一节点
    volatile Node prev;

    // 后一节点
    volatile Node next;

    // 节点所对应的线程
    volatile Thread thread;

    // nextWaiter是“区别当前CLH队列是 ‘独占锁’队列 还是 ‘共享锁’队列 的标记”
    // 若nextWaiter=SHARED，则CLH队列是“独占锁”队列；
    // 若nextWaiter=EXCLUSIVE，(即nextWaiter=null)，则CLH队列是“共享锁”队列。
    Node nextWaiter;

    // “共享锁”则返回true，“独占锁”则返回false。
    final boolean isShared() {
        return nextWaiter == SHARED;
    }

    // 返回前一节点
    final Node predecessor() throws NullPointerException {
        Node p = prev;
        if (p == null)
            throw new NullPointerException();
        else
            return p;
    }

    Node() {    // Used to establish initial head or SHARED marker
    }

    // 构造函数。thread是节点所对应的线程，mode是用来表示thread的锁是“独占锁”还是“共享锁”。
    Node(Thread thread, Node mode) {     // Used by addWaiter
        this.nextWaiter = mode;
        this.thread = thread;
    }

    // 构造函数。thread是节点所对应的线程，waitStatus是线程的等待状态。
    Node(Thread thread, int waitStatus) { // Used by Condition
        this.waitStatus = waitStatus;
        this.thread = thread;
    }
}
```

**说明**：

Node是CLH队列的节点，代表“等待锁的线程队列”。

(01) 每个Node都会一个线程对应。

(02) 每个Node会通过prev和next分别指向上一个节点和下一个节点，这分别代表上一个等待线程和下一个等待线程。

(03) Node通过waitStatus保存线程的等待状态。

(04) Node通过nextWaiter来区分线程是“独占锁”线程还是“共享锁”线程。如果是“独占锁”线程，则nextWaiter的值为EXCLUSIVE；如果是“共享锁”线程，则nextWaiter的值是SHARED。



#### compareAndSetState

```java
/**
     * Atomically sets synchronization state to the given updated
     * value if the current state value equals the expected value.
     * This operation has memory semantics of a {@code volatile} read
     * and write.
     *
     * @param expect the expected value
     * @param update the new value
     * @return {@code true} if successful. False return indicates that the actual
     *         value was not equal to the expected value.
     *
     * 原子地（CAS操作）将同步状态值设置为给定值update如果当前同步状态的值等于expect（期望值）
     */
    protected final boolean compareAndSetState(int expect, int update) {
        // See below for intrinsics setup to support this
        return unsafe.compareAndSwapInt(this, stateOffset, expect, update);
    }
```

#### setExclusiveOwnerThread

```java
/**
 * The current owner of exclusive mode synchronization.
 * exclusiveOwnerThread是当前拥有“独占锁”的线程
 */
private transient Thread exclusiveOwnerThread;

/**
 * Sets the thread that currently owns exclusive access.
 * A {@code null} argument indicates that no thread owns access.
 * This method does not otherwise impose any synchronization or
 * {@code volatile} field accesses.
 * @param thread the owner thread
 *
 * 设置线程t为当前拥有“独占锁”的线程。
 */
protected final void setExclusiveOwnerThread(Thread thread) {
    exclusiveOwnerThread = thread;
}
```



#### setState、getState

state表示锁的状态，对于“独占锁”而已，state=0表示锁是可获取状态(即，锁没有被任何线程锁持有)。由于java中的独占锁是可重入的，state的值可以>1。

```java
// 锁的状态
private volatile int state;
// 设置锁的状态
protected final void setState(int newState) {
    state = newState;
}
// 获取锁的状态
protected final int getState() {
    return state;
}
```



### 2、addWaiter

addWaiter()的作用，就是将当前线程添加到CLH队列中。这就意味着将当前线程添加到等待获取“锁”的等待线程队列中了。

#### addWaiter

对于“公平锁”而言，addWaiter(Node.EXCLUSIVE)会首先创建一个Node节点，节点的类型是“独占锁”(Node.EXCLUSIVE)类型。然后，再将该节点添加到CLH队列的末尾。

```java
/**
     * Creates and enqueues node for current thread and given mode.
     *
     * @param mode Node.EXCLUSIVE for exclusive, Node.SHARED for shared
     * @return the new node
     *
     * 1、将无法获取锁的当前线程封装成Node加入到Sync Queue里面，其中参数mode是独占锁还是共享锁，null表示独占锁。如果是读写锁mode就为共享锁模式
     * 2、Sync Queue实现的一个双向链表，包含head和tail分别指向头部和尾部Node，head和tail设置为了volatile,这两个节点的修改将会被其他线程看到,主要是通过修改这两个节点来完成入队和出队
     * 3、当新加入Node时，将tail指向新加入的Node，同时之前的Node的next指向新Node，新Node的pre指向之前的tail节点，即在双向链表上添加节点成功
     * 4、总而言之，addWaiter的目的就是通过CAS把当前线程封装的Node追加到队尾，并返回该Node实例。
     * 5、把线程要包装为Node对象的主要原因：
     *        a.构造双向链表时，需要指针前驱节点和后驱节点
     *        b.Node中mode用于区分是否是排它模式还是共享模式
     *        c.Node中的waitStatus用于表示当前线程状态：
     * SIGNAL(-1) ：线程的后继线程正/已被阻塞，当该线程release或cancel时要重新唤醒这个后继线程
     * CANCELLED(1)：因为超时或中断，该线程已经被取消，其前驱节点释放锁后不会让处于该种状态的线程去竞争锁资源
     * CONDITION(-2)：表明该线程被处于条件队列，就是因为调用了Condition.await而被阻塞
     * PROPAGATE(-3)：传播共享锁
     * 0：0代表无状态，默认Node就是该种状态，当存在后驱节点追加，就去把其前驱节点设置成SIGNAL
     */
    private Node addWaiter(Node mode) {
        Node node = new Node(Thread.currentThread(), mode);
        // Try the fast path of enq; backup to full enq on failure
        Node pred = tail;
        //默认head = tail = null, tail !=null说明队列中已经有节点,直接CAS到尾节点
        if (pred != null) {
            /**
             * 将当前Node追加到Queue尾部：
             *      1、将当前node的前驱节点设置成tail节点
             *      2、通过cas操作将tail指针指向当前node
             *      3、并将之前的tail节点的next指向当前node
             * 通过上面三步骤，即将一个node追加到双向链表的尾部
             */
            node.prev = pred;
            if (compareAndSetTail(pred, node)) {
                pred.next = node;
                return node;
            }
        }
        //执行到这里，表明Queue队列为空,调用enq会初始化队列并将当前node追加到尾部
        enq(node);
        return node;
    }
```

#### compareAndSetTail

```java
/**
     * CAS tail field. Used only by enq.
     * compareAndSetTail(expect, update)会以原子的方式进行操作，它的作用是判断CLH队列的队尾
     * 是不是为expect，是的话，就将队尾设为update。
     */
    private final boolean compareAndSetTail(Node expect, Node update) {
        return unsafe.compareAndSwapObject(this, tailOffset, expect, update);
    }
```

#### enq

```java
/**
     * Inserts node into queue, initializing if necessary. See picture above.
     * @param node the node to insert
     * @return node's predecessor
     * enq()的作用很简单。如果CLH队列为空，则新建一个CLH表头；然后将node添加到CLH末尾。
     * 否则，直接将node添加到CLH末尾。
     *
     * 1、这里通过一个死循环方式调用CAS，即使有高并发的场景，无限循环将会最终成功把当前线程追加到队尾
     * 2、第一次循环tail肯定为null，则会初始化一个默认的node，并将head=tail指向该node
     * 3、第二次循环的时候，会将当前node追加到1中创建的node尾部
     */
    private Node enq(final Node node) {
        for (;;) {
            Node t = tail;
            if (t == null) { // Must initialize
                if (compareAndSetHead(new Node()))
                    tail = head;
            } else {
                node.prev = t;
                if (compareAndSetTail(t, node)) {
                    t.next = node;
                    return t;
                }
            }
        }
    }
```



### 3、acquireQueued

前面，我们已经将当前线程添加到CLH队列中了。而acquireQueued()的作用就是逐步的去执行CLH队列的线程，如果当前线程获取到了锁，则返回；否则，当前线程进行休眠，直到唤醒并重新获取锁了才返回。下面，我们看看acquireQueued()的具体流程。

```java
/**
     *
     * acquireQueued()的目的是从队列中获取锁。
     * 1、该方法是一个无限死循环，只有保证前驱节点就是头节点，并且重新调用一次tryAcquire()获取锁并成功，才会推送该循环
     * 2、否则会执行shouldParkAfterFailedAcquire()将当前node的前驱节点的waitStatus设置成SIGNAL，表示当它的前驱节点释放锁后会唤醒当前线程，然后当前线程就可以放心的让自己休眠了
     * 3、调用shouldParkAfterFailedAcquire()时，由于默认前驱节点的waitStatus不等于SIGNAL，所以会将前驱节点设置成SIGNAL，但是注意这时的返回结果是false，表示并不会立即让当前线程进入休眠状态，而是重新执行一次for循环，相当于给了一次重新获取锁的机会，如果获取锁成功，则将head节点指向当前节点，之前头结点就废弃了；如果获取失败则调用parkAndCheckInterrupt()让线程真正进入休眠状态
     * 4、parkAndCheckInterrupt()中调用LockSupport.park()让当前线程休眠，客户端也就进入阻塞状态，注意这里有个关键点：当休眠状态的线程被唤醒后，需要再次执行一次for循环通过tryAcquire()来竞争锁资源，竞争成功则退出当前for循环，当然也有可能会竞争失败，如果竞争失败会再次进去休眠状态
     *
     *    Queue队列中的线程是按照从头到尾部的顺序依次唤醒的，每次只会唤醒Queue中的一个线程，为什么还会出现竞争呢？这是因为虽然从Queue中只会唤醒一个线程，但是假如同时又有一个线程执行lock来获取锁资源，而此时并没有放入Queue等待队列中，它就会和从Queue中唤醒的线程进行竞争锁资源，这就体现了非公平锁的特性：后申请锁资源的线程可能会比先申请锁资源的线程优先申请到锁资源。
     *    为什么要这么设计呢？
     *    主要从性能考虑，如果新申请锁的线程可以立即获取到锁，避免了后续一系列创建Node、添加Node到队列等一些列操作，而从Queue中唤醒的线程没有申请到锁只是重新进入休眠，代价要小很多,同时让它们一起竞争锁资源避免Queue等待队列中的线程一直无法获取锁而被饿死情况
     */
    final boolean acquireQueued(final Node node, int arg) {
        // 记录是否获取同步状态成功
        boolean failed = true;
        try {
            // 记录过程中，是否发生线程中断
            boolean interrupted = false;
            for (;;) {
                //1.获取当前节点的前驱节点
                final Node p = node.predecessor();
                //2.判断前驱节点是否是head节点(前继节点是head, 存在两种情况：
                //      a.前继节点现在占用lock
                //      b.前继节点是个空节点,已经释放lock,node现在有机会获取lock; 则再次调用tryAcquire尝试获取一下锁，该源码之前已经分析过
                if (p == head && tryAcquire(arg)) {
                    //3.获取lock成功,直接设置新head(原来的head可能就直接被GC回收)
                    setHead(node);
                    p.next = null; // help GC
                    failed = false;
                    return interrupted;//4.返回在整个获取的过程中是否被中断过；若整个过程中被中断过,则最后我在自我中断一下(selfInterrupt),因为外面的函数可能需要知道整个过程是否被中断过
                }
                //shouldParkAfterFailedAcquire(p, node)返回当前线程是否需要挂起，如果需要则调用parkAndCheckInterrupt()让当前线程休眠
                if (shouldParkAfterFailedAcquire(p, node) &&
                        parkAndCheckInterrupt())//6.parkAndCheckInterrupt会把当前线程挂起，从而阻塞住线程的调用栈,返回值判断是否这次线程的唤醒是被中断唤醒
                    interrupted = true;
            }
        } finally {
            //7.在整个获取中出错
            if (failed)
                //8.清除 node 节点(清除的过程是先给 node 打上 CANCELLED标志, 然后再删除)
                cancelAcquire(node);
        }
    }
```



#### shouldParkAfterFailedAcquire

```java
    /**
     * Checks and updates status for a node that failed to acquire.
     * Returns true if thread should block. This is the main signal
     * control in all acquire loops.  Requires that pred == node.prev.
     *
     * @param pred node's predecessor holding status
     * @param node the node
     * @return {@code true} if thread should block
     */
    private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
        // 获得前一个节点的等待状态
        int ws = pred.waitStatus;
        /**
         * 等待状态为 Node.SIGNAL 时，表示 pred 的下一个节点 node 的线程需要阻塞等待。
         * 在 pred 的线程释放同步状态时，会对 node 的线程进行唤醒通知。所以，返回 true ，
         * 表明当前线程可以被 park，安全的阻塞等待。
         */

        if (ws == Node.SIGNAL)
            /*
             * This node has already set status asking a release
             * to signal it, so it can safely park.
             */
            return true;
        /**
         * 等待状态为 NODE.CANCELLED 时，则表明该线程的前一个节点已经等待超时或者被中断了，
         * 则需要从 CLH 队列中将该前一个节点删除掉，循环回溯，直到前一个节点状态 <= 0 。
         */
        if (ws > 0) {
            /*
             * Predecessor was cancelled. Skip over predecessors and
             * indicate retry.
             */
            do {
                node.prev = pred = pred.prev;
            } while (pred.waitStatus > 0);
            pred.next = node;
        } else {// 0 或者 Node.PROPAGATE
            /*
             * waitStatus must be 0 or PROPAGATE.  Indicate that we
             * need a signal, but don't park yet.  Caller will need to
             * retry to make sure it cannot acquire before parking.
             */
            /**
             * 等待状态为 0 或者 Node.PROPAGATE 时，通过 CAS 设置，将状态修改为 Node.SIGNAL ，
             * 即下一次重新执行 #shouldParkAfterFailedAcquire(Node pred, Node node) 方法时，满足ws == Node.SIGNAL的条件。
             * 但是，对于本次执行，结尾返回 false 。
             * 另外，等待状态不会为 Node.CONDITION ，因为它用在 ConditonObject 中。
             */
            compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
        }
        return false;
    }
```

(01) 关于waitStatus请参考下表(中扩号内为waitStatus的值)，更多关于waitStatus的内容，可以参考前面的Node类的介绍。

```
	CANCELLED[1] -- 当前线程已被取消
​	SIGNAL[-1] -- “当前线程的后继线程需要被unpark(唤醒)”。一般发生情况是：当前线程的后继线程处于阻塞状态，而当前线程被release或cancel掉，因此需要唤醒当前线程的后继线程。
​	CONDITION[-2] -- 当前线程(处在Condition休眠状态)在等待Condition唤醒
​	PROPAGATE[-3] -- (共享锁)其它线程获取到“共享锁”
​	[0] -- 当前线程不属于上面的任何一种状态。
```

(02) shouldParkAfterFailedAcquire()通过以下规则，判断“当前线程”是否需要被阻塞。

```
	规则1：如果前继节点状态为SIGNAL，表明当前节点需要被unpark(唤醒)，此时则返回true。
​	规则2：如果前继节点状态为CANCELLED(ws>0)，说明前继节点已经被取消，则通过先前回溯找到一个有效(非CANCELLED状态)的节点，并返回false。
​	规则3：如果前继节点状态为非SIGNAL、非CANCELLED，则设置前继的状态为SIGNAL，并返回false。
```

如果“规则1”发生，即“前继节点是SIGNAL”状态，则意味着“当前线程”需要被阻塞。接下来会调用parkAndCheckInterrupt()阻塞当前线程，直到当前先被唤醒才从parkAndCheckInterrupt()中返回。



#### parkAndCheckInterrupt

```java
/**
     * Convenience method to park and then check if interrupted
     *
     * @return {@code true} if interrupted
     */
    private final boolean parkAndCheckInterrupt() {
        //利用LockSupport的park方法来挂起当前线程的，直到被唤醒。
        LockSupport.park(this);
        //返回线程的中断状态
        return Thread.interrupted();
    }
```

**说明**：parkAndCheckInterrupt()的作用是阻塞当前线程，并且返回“线程被唤醒之后”的中断状态。

它会先通过LockSupport.park()阻塞“当前线程”，然后通过Thread.interrupted()返回线程的中断状态。

这里介绍一下线程被阻塞之后如何唤醒。一般有2种情况：

**第1种情况**：unpark()唤醒。“前继节点对应的线程”使用完锁之后，通过unpark()方式唤醒当前线程。

**第2种情况**：中断唤醒。其它线程通过interrupt()中断当前线程。

**补充**：LockSupport()中的park(),unpark()的作用 和 Object中的wait(),notify()作用类似，是阻塞/唤醒。

它们的用法不同，park(),unpark()是轻量级的，而wait(),notify()是必须先通过Synchronized获取同步锁。

#### 再次tryAcquire

```java
//1.获取当前节点的前驱节点
                final Node p = node.predecessor();
                //2.判断前驱节点是否是head节点(前继节点是head, 存在两种情况：
                //      a.前继节点现在占用lock
                //      b.前继节点是个空节点,已经释放lock,node现在有机会获取lock; 则再次调用tryAcquire尝试获取一下锁，该源码之前已经分析过
                if (p == head && tryAcquire(arg)) {
                    //3.获取lock成功,直接设置新head(原来的head可能就直接被GC回收)
                    setHead(node);
                    p.next = null; // help GC
                    failed = false;
                    return interrupted;//4.返回在整个获取的过程中是否被中断过；若整个过程中被中断过,则最后我在自我中断一下(selfInterrupt),因为外面的函数可能需要知道整个过程是否被中断过
                }
```

**说明**：

(01) 通过node.predecessor()获取前继节点。predecessor()就是返回node的前继节点，若对此有疑惑可以查看下面关于Node类的介绍。

(02) p == head && tryAcquire(arg)

​    首先，判断“前继节点”是不是CHL表头。如果是的话，则通过tryAcquire()尝试获取锁。

​    其实，这样做的目的是为了“让当前线程获取锁”，但是为什么需要先判断p==head呢？理解这个对理解“公平锁”的机制很重要，因为这么做的原因就是为了保证公平性！

​    (a) 前面，我们在shouldParkAfterFailedAcquire()我们判断“当前线程”是否需要阻塞；

​    (b) 接着，“当前线程”阻塞的话，会调用parkAndCheckInterrupt()来阻塞线程。当线程被解除阻塞的时候，我们会返回线程的中断状态。而线程被解决阻塞，可能是由于“线程被中断”，也可能是由于“其它线程调用了该线程的unpark()函数”。

​    (c) 再回到p==head这里。如果当前线程是因为其它线程调用了unpark()函数而被唤醒，那么唤醒它的线程，应该是它的前继节点所对应的线程(关于这一点，后面在“释放锁”的过程中会看到)。 OK，是前继节点调用unpark()唤醒了当前线程！

​      此时，再来理解p==head就很简单了：当前继节点是CLH队列的头节点，并且它释放锁之后；就轮到当前节点获取锁了。然后，当前节点通过tryAcquire()获取锁；获取成功的话，通过setHead(node)设置当前节点为头节点，并返回。

​    总之，如果“前继节点调用unpark()唤醒了当前线程”并且“前继节点是CLH表头”，此时就是满足p==head，也就是符合公平性原则的。否则，如果当前线程是因为“线程被中断”而唤醒，那么显然就不是公平了。这就是为什么说p==head就是保证公平性！



**小结**：acquireQueued()的作用就是“当前线程”会根据公平性原则进行阻塞等待，直到获取锁为止；并且返回当前线程在等待过程中有没有并中断过。

### 4、selfInterrupt

```java
/**
 * Convenience method to interrupt current thread.
 */
static void selfInterrupt() {
    Thread.currentThread().interrupt();
}
```

**说明**：selfInterrupt()的代码很简单，就是“当前线程”自己产生一个中断。但是，为什么需要这么做呢？

这必须结合acquireQueued()进行分析。如果在acquireQueued()中，当前线程被中断过，则执行selfInterrupt()；否则不会执行。

在acquireQueued()中，即使是线程在阻塞状态被中断唤醒而获取到cpu执行权利；但是，如果该线程的前面还有其它等待锁的线程，根据公平性原则，该线程依然无法获取到锁。它会再次阻塞！ 该线程再次阻塞，直到该线程被它的前面等待锁的线程锁唤醒；线程才会获取锁，然后“真正执行起来”！

也就是说，在该线程“成功获取锁并真正执行起来”之前，它的中断会被忽略并且中断标记会被清除！ 因为在parkAndCheckInterrupt()中，我们线程的中断状态时调用了Thread.interrupted()。该函数不同于Thread的isInterrupted()函数，isInterrupted()仅仅返回中断状态，而interrupted()在返回当前中断状态之后，还会清除中断状态。 正因为之前的中断状态被清除了，所以这里需要调用selfInterrupt()重新产生一个中断！



## 3、释放公平锁

### 1、unlock

```java
		/**
     * Attempts to release this lock.
     *
     * <p>If the current thread is the holder of this lock then the hold
     * count is decremented.  If the hold count is now zero then the lock
     * is released.  If the current thread is not the holder of this
     * lock then {@link IllegalMonitorStateException} is thrown.
     *
     * @throws IllegalMonitorStateException if the current thread does not
     *         hold this lock
     * 由于“公平锁”是可重入的，所以对于同一个线程，每释放锁一次，锁的状态-1。
     */
    public void unlock() {
        sync.release(1);
    }
```

### 2、release

```java
/**
     * Releases in exclusive mode.  Implemented by unblocking one or
     * more threads if {@link #tryRelease} returns true.
     * This method can be used to implement method {@link Lock#unlock}.
     *
     * @param arg the release argument.  This value is conveyed to
     *        {@link #tryRelease} but is otherwise uninterpreted and
     *        can represent anything you like.
     * @return the value returned from {@link #tryRelease}
     *
     * 独占式释放同步状态，该方法会在释放同步状态之后，将同步队列中第一个节点包含的线程唤醒。
     */
    public final boolean release(int arg) {
        if (tryRelease(arg)) {
            Node h = head;
            if (h != null && h.waitStatus != 0)
                unparkSuccessor(h);
            return true;
        }
        return false;
    }
```

### 3、tryRelease

```java
        protected final boolean tryRelease(int releases) {
            // 减掉releases
            int c = getState() - releases;
            // 如果释放的不是持有锁的线程，抛出异常
            if (Thread.currentThread() != getExclusiveOwnerThread())
                throw new IllegalMonitorStateException();
            boolean free = false;
            // state == 0 表示已经释放完全了，其他线程可以获取同步状态了
            if (c == 0) {
                free = true;
                setExclusiveOwnerThread(null);
            }
            setState(c);
            return free;
        }
```

**说明**：

tryRelease()的作用是尝试释放锁。

(01) 如果“当前线程”不是“锁的持有者”，则抛出异常。

(02) 如果“当前线程”在本次释放锁操作之后，对锁的拥有状态是0(即，当前线程彻底释放该“锁”)，则设置“锁”的持有者为null，即锁是可获取状态。同时，更新当前线程的锁的状态为0。

getState(), setState()在前一章已经介绍过，这里不再说明。

getExclusiveOwnerThread(), setExclusiveOwnerThread()在AQS的父类AbstractOwnableSynchronizer.java中定义，源码如下：

```java
public abstract class AbstractOwnableSynchronizer
    implements java.io.Serializable {

    // “锁”的持有线程
    private transient Thread exclusiveOwnerThread;

    // 设置“锁的持有线程”为t
    protected final void setExclusiveOwnerThread(Thread t) {
        exclusiveOwnerThread = t;
    }

    // 获取“锁的持有线程”
    protected final Thread getExclusiveOwnerThread() {
        return exclusiveOwnerThread;
    }
   
    ...
}
```

### 4、unparkSuccessor

在release()中“当前线程”释放锁成功的话，会唤醒当前线程的后继线程。

根据CLH队列的FIFO规则，“当前线程”(即已经获取锁的线程)肯定是head；如果CLH队列非空的话，则唤醒锁的下一个等待线程。

unparkSuccessor()的作用是“唤醒当前线程的后继线程”。后继线程被唤醒之后，就可以获取该锁并恢复运行了。

```java
    /**
     * Wakes up node's successor, if one exists.
     *
     * @param node the node
     */
    private void unparkSuccessor(Node node) {
        /*
         * If status is negative (i.e., possibly needing signal) try
         * to clear in anticipation of signalling.  It is OK if this
         * fails or if status is changed by waiting thread.
         */
        // 获取当前线程的状态
        int ws = node.waitStatus;
        // 如果状态<0，则设置状态=0
        if (ws < 0)
            compareAndSetWaitStatus(node, ws, 0);

        /*
         * Thread to unpark is held in successor, which is normally
         * just the next node.  But if cancelled or apparently null,
         * traverse backwards from tail to find the actual
         * non-cancelled successor.
         */
        //获取当前节点的“有效的后继节点”，无效的话，则通过for循环进行获取。
        // 这里的有效，是指“后继节点对应的线程状态<=0”
        Node s = node.next;
        if (s == null || s.waitStatus > 0) {
            s = null;
            for (Node t = tail; t != null && t != node; t = t.prev)
                if (t.waitStatus <= 0)
                    s = t;
        }
        // 唤醒“后继节点对应的线程”
        if (s != null)
            LockSupport.unpark(s.thread);
    }
```

“释放锁”的过程相对“获取锁”的过程比较简单。释放锁时，主要进行的操作，是更新当前线程对应的锁的状态。如果当前线程对锁已经彻底释放，则设置“锁”的持有线程为null，设置当前线程的状态为空，然后唤醒后继线程。

## 4、获取非公平锁

非公平锁和公平锁在获取锁的方法上，流程是一样的；它们的区别主要表现在“尝试获取锁的机制不同”。简单点说，“公平锁”在每次尝试获取锁时，都是采用公平策略(根据等待队列依次排序等待)；而“非公平锁”在每次尝试获取锁时，都是采用的非公平策略(无视等待队列，直接尝试获取锁，如果锁是空闲的，即可获取状态，则获取锁)。

### 1、lock

```java
/**
         * Performs lock.  Try immediate barge, backing up to normal acquire on failure.
         * state在AbstractQueuedSynchronizer类中定义，表示当前锁状态：
         *
         *  0：当前锁锁未被任何线程持有，线程获取锁资源时可以使用CAS原子操作compareAndSetState(0, 1)
         *     将state由0修改成1，修改成功表示获取锁成功，state这时被修改成1了，并将exclusiveOwnerThread设置成当前Thread，
         *     exclusiveOwnerThread即表示持有锁的线程
         *
         *  >0：表示当前锁被线程持有，因为ReentrantLock是重入锁，同一个线程可以重入多次锁，每重入一次state加1，
         *      同样在释放锁资源release()的时候，每释放一次state减1，直到state=0表示全部释放完成，可以被其它线程竞争使用
         *      state大于0时CAS原子操作compareAndSetState(0, 1)会失败，即进入另一分支
         */
        final void lock() {
            if (compareAndSetState(0, 1))
                //当前线程获取锁成功，并将当前线程赋值给exclusiveOwnerThread变量
                setExclusiveOwnerThread(Thread.currentThread());
            else
                //该分支则表示已有线程持有当前锁
                acquire(1);
        }
```

公平锁 -- 公平锁的lock()函数，会直接调用acquire(1)。

非公平锁 -- 非公平锁会先判断当前锁的状态是不是空闲，是的话，就不排队，而是直接获取锁。

### 2、acquire

公平锁和非公平锁，只有tryAcquire()函数的实现不同；即它们尝试获取锁的机制不同。这就是我们所说的“它们获取锁策略的不同所在之处”！

```java
protected final boolean tryAcquire(int acquires) {
            return nonfairTryAcquire(acquires);
}
```

#### nonfairTryAcquire

```java
/**
         * Performs non-fair tryLock.  tryAcquire is implemented in
         * subclasses, but both need nonfair try for trylock method.
         *
         * 1、该方法重新获取锁state，如果state=0表示当前又没有线程持有该锁了，则重新获取一次锁，成功返回true，失败返回false
         * 2、如果持有该锁的线程就是当前线程，则也会返回true表示获取锁成功，并将state累加acquires
         * 3、否则返回false表示获取锁失败
         */
        final boolean nonfairTryAcquire(int acquires) {
            //当前线程
            final Thread current = Thread.currentThread();
            //获取同步状态
            int c = getState();
            //state == 0,表示没有该锁处于空闲状态
            if (c == 0) {
                //获取锁成功，设置为当前线程所有
                if (compareAndSetState(0, acquires)) {
                    setExclusiveOwnerThread(current);
                    return true;
                }
            }
            //线程重入
            //判断锁持有的线程是否为当前线程
            else if (current == getExclusiveOwnerThread()) {
                //如果持有该锁的线程就是当前线程，则将state+1，然后立即返回true表示获取锁成功，这是可重入锁特性
                int nextc = c + acquires;
                // overflow
                if (nextc < 0)
                    throw new Error("Maximum lock count exceeded");
                //修改state值，此处只会持有锁的线程才会执行，不存在多线程竞争情况，所以通过setState修改，而非CAS，这段代码实现了偏向锁的功能
                setState(nextc);
                return true;
            }
            return false;
        }
```

**说明**：

根据代码，我们可以分析出，tryAcquire()的作用就是尝试去获取锁。

(01) 如果“锁”没有被任何线程拥有，则通过CAS````函数设置“锁”的状态为acquires，同时，设置“当前线程”为锁的持有者，然后返回true。

(02) 如果“锁”的持有者已经是当前线程，则将更新锁的状态即可。

(03) 如果不术语上面的两种情况，则认为尝试失败。

**“公平锁”和“非公平锁”关于tryAcquire()的对比**

*公平锁和非公平锁，它们尝试获取锁的方式不同。*

*公平锁在尝试获取锁时，即使“锁”没有被任何线程锁持有，它也会判断自己是不是CLH等待队列的表头；是的话，才获取锁。*

*而非公平锁在尝试获取锁时，如果“锁”没有被任何线程持有，则不管它在CLH队列的何处，它都直接获取锁。*

## 5、释放非公平锁

与公平锁释放策略相同





## 6、与 synchronized 的区别

前面提到 ReentrantLock 提供了比 synchronized 更加灵活和强大的锁机制，那么它的灵活和强大之处在哪里呢？他们之间又有什么相异之处呢？

首先他们肯定具有相同的功能和内存语义。

与 synchronized 相比，ReentrantLock提供了更多，更加全面的功能，具备更强的扩展性。例如：时间锁等候，可中断锁等候，锁投票。

ReentrantLock 还提供了条件 Condition ，对线程的等待、唤醒操作更加详细和灵活，所以在多个条件变量和高度竞争锁的地方，ReentrantLock 更加适合（以后会阐述Condition）。

ReentrantLock 提供了可轮询的锁请求。它会尝试着去获取锁，如果成功则继续，否则可以等到下次运行时处理，而 synchronized 则一旦进入锁请求要么成功要么阻塞，所以相比 synchronized 而言，ReentrantLock会不容易产生死锁些。

ReentrantLock 支持更加灵活的同步代码块，但是使用 synchronized 时，只能在同一个 synchronized 块结构中获取和释放。注意，ReentrantLock 的锁释放一定要在 finally 中处理，否则可能会产生严重的后果。

ReentrantLock 支持中断处理，且性能较 synchronized 会好些。

 

## 7、总结

公平锁和非公平锁的区别，是在获取锁的机制上的区别。表现在，在尝试获取锁时 —— 公平锁，只有在当前线程是CLH等待队列的表头时，才获取锁；而非公平锁，只要当前锁处于空闲状态，则直接获取锁，而不管CLH等待队列中的顺序。

只有当非公平锁尝试获取锁失败的时候，它才会像公平锁一样，进入CLH等待队列排序等待。




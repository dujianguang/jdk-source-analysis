# AbstractQueuedSynchronizer

## 简介

AQS ，AbstractQueuedSynchronizer ，即**队列同步器**。它是构建锁或者其他同步组件的基础框架（如 ReentrantLock、ReentrantReadWriteLock、Semaphore 等），J.U.C 并发包的作者（**Doug Lea**）期望它能够成为实现大部分同步需求的基础。

它是 J.U.C 并发包中的核心基础组件。



AQS 解决了在实现同步器时涉及当的大量细节问题，例如获取同步状态、FIFO 同步队列。基于 AQS 来构建同步器可以带来**很多好处**。它不仅能够极大地减少实现工作，而且也不必处理在多个位置上发生的竞争问题。

在基于 AQS 构建的同步器中，只能在一个时刻发生阻塞，从而降低上下文切换的开销，提高了吞吐量。同时在设计 AQS 时充分考虑了可伸缩性，因此 J.U.C 中，所有基于 AQS 构建的同步器均可以获得这个优势。

**AQS核心思想是，如果被请求的共享资源空闲，则将当前请求资源的线程设置为有效的工作线程，并且将共享资源设置为锁定状态。如果被请求的共享资源被占用，那么就需要一套线程阻塞等待以及被唤醒时锁分配的机制，这个机制AQS是用CLH队列锁实现的，即将暂时获取不到锁的线程加入到队列中。**

> CLH(Craig,Landin,and Hagersten)队列是一个虚拟的双向队列（虚拟的双向队列即不存在队列实例，仅存在结点之间的关联关系）。AQS是将每条请求共享资源的线程封装成一个CLH锁队列的一个结点（Node）来实现锁的分配。

\2. **AQS**锁的类别 -- 分为“**独占锁**”和“**共享锁**”两种。

  (01) **独占锁** -- 锁在一个时间点只能被一个线程锁占有。根据锁的获取机制，它又划分为“**公平锁**”和“**非公平锁**”。公平锁，是按照通过CLH等待线程按照先来先得的规则，公平的获取锁；而非公平锁，则当线程要获取锁时，它会无视CLH等待队列而直接获取锁。独占锁的典型实例子是ReentrantLock，此外，ReentrantReadWriteLock.WriteLock也是独占锁。

  (02) **共享锁** -- 能被多个线程同时拥有，能被共享的锁。JUC包中的ReentrantReadWriteLock.ReadLock，CyclicBarrier， CountDownLatch和Semaphore都是共享锁。这些锁的用途和原理，在以后的章节再详细介绍。

\3. **CLH队列** -- Craig, Landin, and Hagersten lock queue

  CLH队列是AQS中“等待锁”的线程队列。在多线程中，为了保护竞争资源不被多个线程同时操作而起来错误，我们常常需要通过锁来保护这些资源。在独占锁中，竞争资源在一个时间点只能被一个线程锁访问；而其它线程则需要等待。CLH就是管理这些“等待锁”的线程的队列。

  CLH是一个非阻塞的 FIFO 队列。也就是说往里面插入或移除一个节点的时候，在并发条件下不会阻塞，而是通过自旋锁和 CAS 保证节点插入和移除的原子性。

\4. **CAS函数** -- Compare And Swap 

  CAS函数，是比较并交换函数，它是原子操作函数；即，通过CAS操作的数据都是以**原子方式**进行的。例如，compareAndSetHead(), compareAndSetTail(), compareAndSetNext()等函数。它们共同的特点是，这些函数所执行的动作是以原子的方式进行的。

## 同步状态

AQS 的主要使用方式是**继承**，子类通过继承同步器，并实现它的**抽象方法**来管理同步状态。

AQS 使用一个 int 类型的成员变量 state 来**表示同步状态**：

当 state > 0 时，表示已经获取了锁。

当 state = 0 时，表示释放了锁。

它提供了三个方法，来对同步状态 state 进行操作，并且 AQS 可以确保对 state 的操作是**安全**的：

\#getState()

\#setState(int newState)

\#compareAndSetState(int expect, int update)



## 同步队列

AQS 通过内置的 FIFO 同步队列来完成资源获取线程的**排队工作**：

如果当前线程获取同步状态失败（锁）时，AQS 则会将当前线程以及等待状态等信息构造成一个节点（Node）并将其加入同步队列，同时会阻塞当前线程

当同步状态**释放**时，则会把节点中的线程唤醒，使其再次尝试获取同步状态。



## 核心内置方法

基本上可以分成 **3** 类：

**独占式**获取与释放同步状态

**共享式**获取与释放同步状态

查询**同步队列**中的等待线程情况





自定义子类使用 AQS 提供的模板方法，就可以实现**自己的同步语义**。



AQS使用了模板方法模式，自定义同步器时需要重写下面几个AQS提供的模板方法：

isHeldExclusively()*//该线程是否正在独占资源。只有用到condition才需要去实现它。*

tryAcquire(int)*//独占方式。尝试获取资源，成功则返回true，失败则返回false。*

tryRelease(int)*//独占方式。尝试释放资源，成功则返回true，失败则返回false。*

tryAcquireShared(int)*//共享方式。尝试获取资源。负数表示失败；0表示成功，但没有剩余可用资源；正数表示成功，且有剩余资源。*

tryReleaseShared(int)//共享方式。尝试释放资源，成功则返回true，失败则返回false。

默认情况下，每个方法都抛出 UnsupportedOperationException。 这些方法的实现必须是内部线程安全的，并且通常应该简短而不是阻塞。AQS类中的其他方法都是final ，所以无法被其他类使用，只有这几个方法可以被其他类使用。

以ReentrantLock为例，state初始化为0，表示未锁定状态。A线程lock()时，会调用tryAcquire()独占该锁并将state+1。此后，其他线程再tryAcquire()时就会失败，直到A线程unlock()到state=0（即释放锁）为止，其它线程才有机会获取该锁。当然，释放锁之前，A线程自己是可以重复获取此锁的（state会累加），这就是可重入的概念。但要注意，获取多少次就要释放多么次，这样才能保证state是能回到零态的。

再以CountDownLatch以例，任务分为N个子线程去执行，state也初始化为N（注意N要与线程个数一致）。这N个子线程是并行执行的，每个子线程执行完后countDown()一次，state会CAS(Compare and Swap)减1。等到所有子线程都执行完后(即state=0)，会unpark()主调用线程，然后主调用线程就会从await()函数返回，继续后余动作。

一般来说，自定义同步器要么是独占方法，要么是共享方式，他们也只需实现tryAcquire-tryRelease、tryAcquireShared-tryReleaseShared中的一种即可。但AQS也支持自定义同步器同时实现独占和共享两种方式，如ReentrantReadWriteLock。



## 独占式

独占式，**同一时刻，仅有一个线程持有同步状态**。



### 1、独占式同步状态获取

#### acquire

```
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



#### acquireQueued

```
/**
     * Acquires in exclusive uninterruptible mode for thread already in
     * queue. Used by condition wait methods as well as acquire.
     *
     * @param node the node
     * @param arg the acquire argument
     * @return {@code true} if interrupted while waiting
     */
    /**
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

####  shouldParkAfterFailedAcquire

```
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

#### cancelAcquire

```
/**
     * Cancels an ongoing attempt to acquire.
     *
     * @param node the node
     */
    private void cancelAcquire(Node node) {
        // Ignore if node doesn't exist
        if (node == null)
            return;
        //将节点的等待线程置空。
        node.thread = null;

        // Skip cancelled predecessors
        //获得 node 节点的前一个节点 pred 。
        Node pred = node.prev;
        while (pred.waitStatus > 0)
            node.prev = pred = pred.prev;

        // predNext is the apparent node to unsplice. CASes below will
        // fail if not, in which case, we lost race vs another cancel
        // or signal, so no further action is necessary.
        Node predNext = pred.next;

        // Can use unconditional write instead of CAS here.
        // After this atomic step, other Nodes can skip past us.
        // Before, we are free of interference from other threads.
        node.waitStatus = Node.CANCELLED;

        // If we are the tail, remove ourselves.
        if (node == tail && compareAndSetTail(node, pred)) {
            compareAndSetNext(pred, predNext, null);
        } else {
            // If successor needs signal, try to set pred's next-link
            // so it will get one. Otherwise wake it up to propagate.
            int ws;
            if (pred != head &&
                ((ws = pred.waitStatus) == Node.SIGNAL ||
                 (ws <= 0 && compareAndSetWaitStatus(pred, ws, Node.SIGNAL))) &&
                pred.thread != null) {
                Node next = node.next;
                if (next != null && next.waitStatus <= 0)
                    compareAndSetNext(pred, predNext, next);
            } else {
                unparkSuccessor(node);
            }

            node.next = node; // help GC
        }
    }
```



### 2、独占式获取响应中断

#### acquireInterruptibly

#### doAcquireInterruptibly





### 3、独占式超时获取

#### tryAcquireNanos









### 4、独占式同步状态释放





### 5、总结

在 AQS 中维护着一个 FIFO 的同步队列。

当线程获取同步状态失败后，则会加入到这个 CLH 同步队列的对尾，并一直保持着自旋。

在 CLH 同步队列中的线程在自旋时，会判断其前驱节点是否为首节点，如果为首节点则不断尝试获取同步状态，获取成功则退出CLH同步队列。

当线程执行完逻辑后，会释放同步状态，释放后会唤醒其后继节点。

## 共享式

共享式与独占式的最主要区别在于，**同一时刻**：

独占式只能有**一个**线程获取同步状态。

共享式可以有**多个**线程获取同步状态。

例如，读操作可以有多个线程同时进行，而写操作同一时刻只能有一个线程进行写操作，其他操作都会被阻塞。参见 ReentrantReadWriteLock 。

### 1、共享式同步状态获取

### 2、共享式获取响应中断

### 3、 共享式超时获取

### 4、共享式同步状态释放


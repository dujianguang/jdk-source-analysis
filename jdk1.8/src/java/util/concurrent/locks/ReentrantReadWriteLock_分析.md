# ReentrantReadWriteLock

重入锁 ReentrantLock 是排他锁，**排他锁在同一时刻仅有一个线程可以进行访问**，但是在大多数场景下，大部分时间都是提供读服务，而写服务占有的时间较少。然而，读服务不存在数据竞争问题，如果一个线程在读时禁止其他线程读势必会导致性能降低。所以就提供了读写锁。

读写锁维护着**一对**锁，一个读锁和一个写锁。通过分离读锁和写锁，使得并发性比一般的排他锁有了较大的提升：

在同一时间，可以允许**多个**读线程同时访问。

但是，在写线程访问时，所有读线程和写线程都会被阻塞。

读写锁的**主要特性**：

公平性：支持公平性和非公平性。

重入性：支持重入。读写锁最多支持 65535 个递归写入锁和 65535 个递归读取锁。

锁降级：遵循获取**写**锁，再获取**读**锁，最后释放**写**锁的次序，如此写锁能够**降级**成为读锁。

## ReadWriteLock

```java
/**
 * 分别获得读锁写锁对象
 */
public interface ReadWriteLock {
    /**
     * Returns the lock used for reading.
     *
     * @return the lock used for reading
     */
    Lock readLock();

    /**
     * Returns the lock used for writing.
     *
     * @return the lock used for writing
     */
    Lock writeLock();
}
```

#### 读写锁状态划分方式

<img src="https://github.com/muyutingfeng/jdk-source-analysis/raw/master/note/doc/java/util/concurrent/locks/ReentrantReadWriteLock/%E8%AF%BB%E5%86%99%E9%94%81%E7%8A%B6%E6%80%81%E5%88%92%E5%88%86%E6%96%B9%E5%BC%8F.png?raw=true" alt="ReentrantReadWriteLock" style="zoom:50%;" />

## ReentrantReadWriteLock

### getThreadId

```java
		/**
     * Returns the thread id for the given thread.  We must access
     * this directly rather than via method Thread.getId() because
     * getId() is not final, and has been known to be overridden in
     * ways that do not preserve unique mappings.
     *
     * 获得线程编号
     *
     * Thread#getId()非final修饰的，如果有实现 Thread 的子类，完全可以覆写这个方法，所以可能导致无法获得 tid 属性。
     * 因此上面的方法，使用 Unsafe 直接获得 tid 属性。
     *
     * java.lang.Thread.getId() should be final
     * https://bugs.openjdk.java.net/browse/JDK-6346938
     */
    static final long getThreadId(Thread thread) {
        return UNSAFE.getLongVolatile(thread, TID_OFFSET);
    }
```



### 读锁和写锁

#### ReadLock

ReadLock 是 ReentrantReadWriteLock 的内部静态类，实现 java.util.concurrent.locks.Lock 接口，读锁实现类。

##### 构造方法

```java
				/**
         * Constructor for use by subclasses
         *
         * sync 字段，通过 ReentrantReadWriteLock 的构造方法，传入并使用它的 Sync 对象。
         * @param lock the outer lock object
         * @throws NullPointerException if the lock is null
         *
         */
        protected ReadLock(ReentrantReadWriteLock lock) {
            sync = lock.sync;
        }
```

##### lock

```java
/**
         * Acquires the read lock.
         *
         * <p>Acquires the read lock if the write lock is not held by
         * another thread and returns immediately.
         *
         * <p>If the write lock is held by another thread then
         * the current thread becomes disabled for thread scheduling
         * purposes and lies dormant until the read lock has been acquired.
         *
         * 调用 AQS 的 #acquireShared(int arg) 方法，共享式获得同步状态。
         * 所以，读锁可以同时被多个线程获取。
         */
        public void lock() {
            sync.acquireShared(1);
        }
```

##### lockInterruptibly

```java
    /**
         * Acquires the read lock unless the current thread is
         * {@linkplain Thread#interrupt interrupted}.
         *
         * <p>Acquires the read lock if the write lock is not held
         * by another thread and returns immediately.
         *
         * <p>If the write lock is held by another thread then the
         * current thread becomes disabled for thread scheduling
         * purposes and lies dormant until one of two things happens:
         *
         * <ul>
         *
         * <li>The read lock is acquired by the current thread; or
         *
         * <li>Some other thread {@linkplain Thread#interrupt interrupts}
         * the current thread.
         *
         * </ul>
         *
         * <p>If the current thread:
         *
         * <ul>
         *
         * <li>has its interrupted status set on entry to this method; or
         *
         * <li>is {@linkplain Thread#interrupt interrupted} while
         * acquiring the read lock,
         *
         * </ul>
         *
         * then {@link InterruptedException} is thrown and the current
         * thread's interrupted status is cleared.
         *
         * <p>In this implementation, as this method is an explicit
         * interruption point, preference is given to responding to
         * the interrupt over normal or reentrant acquisition of the
         * lock.
         *
         * @throws InterruptedException if the current thread is interrupted
         */
        public void lockInterruptibly() throws InterruptedException {
            sync.acquireSharedInterruptibly(1);
        }
```

##### tryLock

\#tryLock() **实现**方法，在实现时，希望能**快速的**获得是否能够获得到锁，因此即使在设置为 fair = true ( 使用公平锁 )，依然调用 Sync#tryReadLock() 方法。

如果**真的**希望 #tryLock() 还是按照是否公平锁的方式来，可以调用 #tryLock(0, TimeUnit) 方法来实现。

```java
        public boolean tryLock() {
            return sync.tryReadLock();
        }
        
        
        public boolean tryLock(long timeout, TimeUnit unit)
                throws InterruptedException {
            return sync.tryAcquireSharedNanos(1, unit.toNanos(timeout));
        }
```



##### unlock

```java
/**
         * Attempts to release this lock.
         *
         * <p>If the number of readers is now zero then the lock
         * is made available for write lock attempts.
         *
         * 调用 AQS 的 #releaseShared(int arg) 方法，共享式释放同步状态。
         */
        public void unlock() {
            sync.releaseShared(1);
        }
```

newCondition

```java
        /**
         * Throws {@code UnsupportedOperationException} because
         * {@code ReadLocks} do not support conditions.
         *
         * 不支持？？
         * @throws UnsupportedOperationException always
         */
        public Condition newCondition() {
            throw new UnsupportedOperationException();
        }
```



#### WriteLock

WriteLock 的代码，类似 ReadLock 的代码，差别在于**独占式**获取同步状态。

WriteLock 是 ReentrantReadWriteLock 的内部静态类，实现 java.util.concurrent.locks.Lock 接口，写锁实现类。

##### 构造方法

```java
        /**
         * Constructor for use by subclasses
         *
         * @param lock the outer lock object
         * @throws NullPointerException if the lock is null
         */
        protected WriteLock(ReentrantReadWriteLock lock) {
            sync = lock.sync;
        }
```

##### lock

```java
        /**
         * Acquires the write lock.
         *
         * <p>Acquires the write lock if neither the read nor write lock
         * are held by another thread
         * and returns immediately, setting the write lock hold count to
         * one.
         *
         * <p>If the current thread already holds the write lock then the
         * hold count is incremented by one and the method returns
         * immediately.
         *
         * <p>If the lock is held by another thread then the current
         * thread becomes disabled for thread scheduling purposes and
         * lies dormant until the write lock has been acquired, at which
         * time the write lock hold count is set to one.
         *
         * 调用 AQS 的 #.acquire(int arg) 方法，独占式获得同步状态。
         * 所以，写锁只能同时被一个线程获取。
         */
        public void lock() {
            sync.acquire(1);
        }
```

##### lockInterruptibly

```java
/**
         * Acquires the write lock unless the current thread is
         * {@linkplain Thread#interrupt interrupted}.
         *
         * <p>Acquires the write lock if neither the read nor write lock
         * are held by another thread
         * and returns immediately, setting the write lock hold count to
         * one.
         *
         * <p>If the current thread already holds this lock then the
         * hold count is incremented by one and the method returns
         * immediately.
         *
         * <p>If the lock is held by another thread then the current
         * thread becomes disabled for thread scheduling purposes and
         * lies dormant until one of two things happens:
         *
         * <ul>
         *
         * <li>The write lock is acquired by the current thread; or
         *
         * <li>Some other thread {@linkplain Thread#interrupt interrupts}
         * the current thread.
         *
         * </ul>
         *
         * <p>If the write lock is acquired by the current thread then the
         * lock hold count is set to one.
         *
         * <p>If the current thread:
         *
         * <ul>
         *
         * <li>has its interrupted status set on entry to this method;
         * or
         *
         * <li>is {@linkplain Thread#interrupt interrupted} while
         * acquiring the write lock,
         *
         * </ul>
         *
         * then {@link InterruptedException} is thrown and the current
         * thread's interrupted status is cleared.
         *
         * <p>In this implementation, as this method is an explicit
         * interruption point, preference is given to responding to
         * the interrupt over normal or reentrant acquisition of the
         * lock.
         *
         * @throws InterruptedException if the current thread is interrupted
         */
        public void lockInterruptibly() throws InterruptedException {
            sync.acquireInterruptibly(1);
        }
```

##### tryLock

```java
        /**
         * Acquires the write lock only if it is not held by another thread
         * at the time of invocation.
         *
         * <p>Acquires the write lock if neither the read nor write lock
         * are held by another thread
         * and returns immediately with the value {@code true},
         * setting the write lock hold count to one. Even when this lock has
         * been set to use a fair ordering policy, a call to
         * {@code tryLock()} <em>will</em> immediately acquire the
         * lock if it is available, whether or not other threads are
         * currently waiting for the write lock.  This &quot;barging&quot;
         * behavior can be useful in certain circumstances, even
         * though it breaks fairness. If you want to honor the
         * fairness setting for this lock, then use {@link
         * #tryLock(long, TimeUnit) tryLock(0, TimeUnit.SECONDS) }
         * which is almost equivalent (it also detects interruption).
         *
         * <p>If the current thread already holds this lock then the
         * hold count is incremented by one and the method returns
         * {@code true}.
         *
         * <p>If the lock is held by another thread then this method
         * will return immediately with the value {@code false}.
         *
         * @return {@code true} if the lock was free and was acquired
         * by the current thread, or the write lock was already held
         * by the current thread; and {@code false} otherwise.
         *
         * #tryLock() 实现方法，在实现时，希望能快速的获得是否能够获得到锁，因此即使在设置为 fair = true ( 使用公平锁 )，依然调用 Sync#tryWriteLock() 方法。
         * 如果真的希望 #tryLock() 还是按照是否公平锁的方式来，可以调用 #tryLock(0, TimeUnit) 方法来实现。
         */
        public boolean tryLock( ) {
            return sync.tryWriteLock();
        }
        
        public boolean tryLock(long timeout, TimeUnit unit) throws InterruptedException {
            return sync.tryAcquireNanos(1, unit.toNanos(timeout));
        }
```

##### unlock

```java
        /**
         * Attempts to release this lock.
         *
         * <p>If the current thread is the holder of this lock then
         * the hold count is decremented. If the hold count is now
         * zero then the lock is released.  If the current thread is
         * not the holder of this lock then {@link
         * IllegalMonitorStateException} is thrown.
         *
         * @throws IllegalMonitorStateException if the current thread does not
         * hold this lock
         *
         * 调用 AQS 的 #release(int arg) 方法，独占式释放同步状态。
         */
        public void unlock() {
            sync.release(1);
        }
```

##### newCondition

```java
        /**
         * Returns a {@link Condition} instance for use with this
         * {@link Lock} instance.
         * <p>The returned {@link Condition} instance supports the same
         * usages as do the {@link Object} monitor methods ({@link
         * Object#wait() wait}, {@link Object#notify notify}, and {@link
         * Object#notifyAll notifyAll}) when used with the built-in
         * monitor lock.
         *
         * <ul>
         *
         * <li>If this write lock is not held when any {@link
         * Condition} method is called then an {@link
         * IllegalMonitorStateException} is thrown.  (Read locks are
         * held independently of write locks, so are not checked or
         * affected. However it is essentially always an error to
         * invoke a condition waiting method when the current thread
         * has also acquired read locks, since other threads that
         * could unblock it will not be able to acquire the write
         * lock.)
         *
         * <li>When the condition {@linkplain Condition#await() waiting}
         * methods are called the write lock is released and, before
         * they return, the write lock is reacquired and the lock hold
         * count restored to what it was when the method was called.
         *
         * <li>If a thread is {@linkplain Thread#interrupt interrupted} while
         * waiting then the wait will terminate, an {@link
         * InterruptedException} will be thrown, and the thread's
         * interrupted status will be cleared.
         *
         * <li> Waiting threads are signalled in FIFO order.
         *
         * <li>The ordering of lock reacquisition for threads returning
         * from waiting methods is the same as for threads initially
         * acquiring the lock, which is in the default case not specified,
         * but for <em>fair</em> locks favors those threads that have been
         * waiting the longest.
         *
         * </ul>
         *
         * @return the Condition object
         */
        public Condition newCondition() {
            return sync.newCondition();
        }
```

##### isHeldByCurrentThread

```java
        /**
         * Queries if this write lock is held by the current thread.
         * Identical in effect to {@link
         * ReentrantReadWriteLock#isWriteLockedByCurrentThread}.
         *
         * @return {@code true} if the current thread holds this lock and
         *         {@code false} otherwise
         * @since 1.6
         *
         * 判断是否被当前线程独占锁。
         */
        public boolean isHeldByCurrentThread() {
            return sync.isHeldExclusively();
        }
```

##### getHoldCount

```java
        /**
         * Queries the number of holds on this write lock by the current
         * thread.  A thread has a hold on a lock for each lock action
         * that is not matched by an unlock action.  Identical in effect
         * to {@link ReentrantReadWriteLock#getWriteHoldCount}.
         *
         * @return the number of holds on this lock by the current thread,
         *         or zero if this lock is not held by the current thread
         * @since 1.6
         *
         * 返回当前线程独占锁的持有数量
         */
        public int getHoldCount() {
            return sync.getWriteHoldCount();
        }
```



### Sync抽象类



#### readerShouldBlock

```java
        /**
         * Returns true if the current thread, when trying to acquire
         * the read lock, and otherwise eligible to do so, should block
         * because of policy for overtaking other waiting threads.
         */
        abstract boolean readerShouldBlock();
```



#### writerShouldBlock

```java
        /**
         * Returns true if the current thread, when trying to acquire
         * the write lock, and otherwise eligible to do so, should block
         * because of policy for overtaking other waiting threads.
         *
         * 获取写锁时，如果有前序节点也获得锁时，是否阻塞。NonefairSync 和 FairSync 下有不同的实现。详细解析，见 「6. Sync 实现类」 。
         */
        abstract boolean writerShouldBlock();
```



#### tryAcquire

```java
protected final boolean tryAcquire(int acquires) {
            /*
             * Walkthrough:
             * 1. If read count nonzero or write count nonzero
             *    and owner is a different thread, fail.
             * 2. If count would saturate, fail. (This can only
             *    happen if count is already nonzero.)
             * 3. Otherwise, this thread is eligible for lock if
             *    it is either a reentrant acquire or
             *    queue policy allows it. If so, update state
             *    and set owner.
             */
            Thread current = Thread.currentThread();
            //当前锁个数
            int c = getState();
            //写锁
            int w = exclusiveCount(c);
            if (c != 0) {
                // (Note: if c != 0 and w == 0 then shared count != 0)
                //c != 0 && w == 0 表示存在读锁
                //当前线程不是已经获取写锁的线程
                if (w == 0 || current != getExclusiveOwnerThread())
                    return false;
                //超出最大范围
                if (w + exclusiveCount(acquires) > MAX_COUNT)
                    throw new Error("Maximum lock count exceeded");
                // Reentrant acquire
                setState(c + acquires);
                return true;
            }
            // 是否需要阻塞
            if (writerShouldBlock() ||
                !compareAndSetState(c, c + acquires))
                return false;
            //设置获取锁的线程为当前线程
            setExclusiveOwnerThread(current);
            return true;
        }
```

该方法和 ReentrantLock 的 #tryAcquire(int arg) **大致一样**，差别在**判断重入**时，增加了一项条件：读锁是否存在。因为要确保写锁的操作对读锁是**可见的**。如果在存在读锁的情况下允许获取写锁，那么那些已经获取读锁的其他线程可能就无法感知当前写线程的操作。因此只有等读锁完全释放后，写锁才能够被当前线程所获取，一旦写锁获取了，所有其他读、写线程均会被阻塞。

调用 #writerShouldBlock() **抽象**方法，若返回 true ，则获取写锁**失败**。

#### tryAcquireShared

\#tryAcqurireShared(int arg) 方法，尝试获取**读同步状态**，获取成功返回 >= 0 的结果，否则返回 < 0 的结果。代码如下：

```java
protected final int tryAcquireShared(int unused) {
            /*
             * Walkthrough:
             * 1. If write lock held by another thread, fail.
             * 2. Otherwise, this thread is eligible for
             *    lock wrt state, so ask if it should block
             *    because of queue policy. If not, try
             *    to grant by CASing state and updating count.
             *    Note that step does not check for reentrant
             *    acquires, which is postponed to full version
             *    to avoid having to check hold count in
             *    the more typical non-reentrant case.
             * 3. If step 2 fails either because thread
             *    apparently not eligible or CAS fails or count
             *    saturated, chain to version with full retry loop.
             *
             *
             *
             *    为何要引入 firstReader、firstReaderHoldCount 变量。这是为了一个效率问题，
             *    firstReader 是不会放入到 readHolds 中的，如果读锁仅有一个的情况下，
             *    就会避免查找 readHolds 。
             *
             *
             *    锁降级中读锁的获取释放为必要？肯定是必要的。试想，假如当前线程 A 不获取读锁而是直接
             *    释放了写锁，这个时候另外一个线程 B 获取了写锁，那么这个线程 B 对数据的修改是不会对
             *    当前线程 A 可见的。如果获取了读锁，则线程B在获取写锁过程中判断如果有读锁还没有释放
             *    则会被阻塞，只有当前线程 A 释放读锁后，线程 B 才会获取写锁成功。
             */
            Thread current = Thread.currentThread();
            //exclusiveCount(c)计算写锁
            //如果存在写锁，且锁的持有者不是当前线程，直接返回-1
            //存在锁降级问题，后续阐述
            int c = getState();
            if (exclusiveCount(c) != 0 &&
                getExclusiveOwnerThread() != current)
                return -1;
            //读锁
            int r = sharedCount(c);
            /*
             * readerShouldBlock():读锁是否需要等待（公平锁原则）
             * r < MAX_COUNT：持有线程小于最大数（65535）
             * compareAndSetState(c, c + SHARED_UNIT)：设置读取锁状态
             */
            if (!readerShouldBlock() &&
                r < MAX_COUNT &&
                compareAndSetState(c, c + SHARED_UNIT)) {//修改高16位的状态，所以要加上2^16
                if (r == 0) {
                    firstReader = current;
                    firstReaderHoldCount = 1;
                //如果获取读锁的线程为第一次获取读锁的线程，则firstReaderHoldCount重入数 + 1
                } else if (firstReader == current) {
                    firstReaderHoldCount++;
                } else {
                    HoldCounter rh = cachedHoldCounter;
                    //rh == null 或者 rh.tid != current.getId()，需要获取rh
                    if (rh == null || rh.tid != getThreadId(current))
                        cachedHoldCounter = rh = readHolds.get();
                    else if (rh.count == 0)
                        //加入到readHolds中
                        readHolds.set(rh);
                    rh.count++;
                }
                return 1;
            }
            return fullTryAcquireShared(current);
        }
```

读锁获取的过程相对于独占锁而言会稍微复杂下，整个过程如下：

因为存在锁降级情况，如果存在写锁且锁的持有者不是当前线程，则直接返回失败，否则继续。

依据公平性原则，调用 #readerShouldBlock() 方法来判断读锁是否**不需**要阻塞，读锁持有线程数小于最大值（65535），且 **CAS** 设置锁状态成功，执行代码，并返回 1 。如果不满足任一条件，则调用 #fullTryAcquireShared(Thread thread) 方法。

##### fullTryAcquireShared

```java
        /**
         * Full version of acquire for reads, that handles CAS misses
         * and reentrant reads not dealt with in tryAcquireShared.
         */
        final int fullTryAcquireShared(Thread current) {
            /*
             * This code is in part redundant with that in
             * tryAcquireShared but is simpler overall by not
             * complicating tryAcquireShared with interactions between
             * retries and lazily reading hold counts.
             */
            HoldCounter rh = null;
            for (;;) {
                int c = getState();
                // 锁降级
                if (exclusiveCount(c) != 0) {
                    if (getExclusiveOwnerThread() != current)
                        return -1;
                    // else we hold the exclusive lock; blocking here
                    // would cause deadlock.
                    // 读锁需要阻塞，判断是否当前线程已经获取到读锁
                } else if (readerShouldBlock()) {
                    // Make sure we're not acquiring read lock reentrantly
                    //列头为当前线程
                    if (firstReader == current) {
                        // assert firstReaderHoldCount > 0;
                    } else {
                        if (rh == null) {
                            rh = cachedHoldCounter;
                            if (rh == null || rh.tid != getThreadId(current)) {
                                rh = readHolds.get();
                                // 计数为 0 ，说明没得到读锁，清空线程变量
                                if (rh.count == 0)
                                    readHolds.remove();
                            }
                        }
                        if (rh.count == 0)
                            return -1;
                    }
                }
                //读锁超出最大范围
                if (sharedCount(c) == MAX_COUNT)
                    throw new Error("Maximum lock count exceeded");
                //CAS设置读锁成功
                //修改高16位的状态，所以要加上2^16
                if (compareAndSetState(c, c + SHARED_UNIT)) {
                    //如果是第1次获取“读取锁”，则更新firstReader和firstReaderHoldCount
                    if (sharedCount(c) == 0) {
                        //如果想要获取锁的线程(current)是第1个获取锁(firstReader)的线程,则将firstReaderHoldCount+1
                        firstReader = current;
                        firstReaderHoldCount = 1;
                    } else if (firstReader == current) {
                        firstReaderHoldCount++;
                    } else {
                        if (rh == null)
                            rh = cachedHoldCounter;
                        if (rh == null || rh.tid != getThreadId(current))
                            rh = readHolds.get();
                        // 说明没得到读锁
                        else if (rh.count == 0)
                            readHolds.set(rh);
                        //更新线程的获取“读取锁”的共享计数
                        rh.count++;
                        // cache for release
                        cachedHoldCounter = rh;
                    }
                    return 1;
                }
            }
        }
```

该方法会根据“是否需要阻塞等待”，“读取锁的共享计数是否超过限制”等等进行处理。如果不需要阻塞等待，并且锁的共享计数没有超过限制，则通过 **CAS** 尝试获取锁，并返回 1 。所以，#fullTryAcquireShared(Thread) 方法，是 #tryAcquireShared(int unused) 方法的**自旋重试的**逻辑。

#### tryRelease

```java
/*
         * Note that tryRelease and tryAcquire can be called by
         * Conditions. So it is possible that their arguments contain
         * both read and write holds that are all released during a
         * condition wait and re-established in tryAcquire.
         *
         * 写锁释放锁的整个过程，和独占锁 ReentrantLock 相似，每次释放均是减少写状态，当写状态为 0 时，
         * 表示写锁已经完全释放了，从而让等待的其他线程可以继续访问读、写锁，获取同步状态。同时，
         * 此次写线程的修改对后续的线程可见。
         */
        protected final boolean tryRelease(int releases) {
            //释放的线程不为锁的持有者
            if (!isHeldExclusively())
                throw new IllegalMonitorStateException();
            int nextc = getState() - releases;
            //若写锁的新线程数为0，则将锁的持有者设置为null
            boolean free = exclusiveCount(nextc) == 0;
            if (free)
                setExclusiveOwnerThread(null);
            setState(nextc);
            return free;
        }
```

#### tryReleaseShared

```java
protected final boolean tryReleaseShared(int unused) {
            Thread current = Thread.currentThread();
            //如果想要释放锁的线程为第一个获取锁的线程
            if (firstReader == current) {
                //仅获取了一次，则需要将firstReader 设置null，否则 firstReaderHoldCount - 1
                if (firstReaderHoldCount == 1)
                    firstReader = null;
                else
                    firstReaderHoldCount--;
            }
            //获取rh对象，并更新“当前线程获取锁的信息”
            else {
                HoldCounter rh = cachedHoldCounter;
                if (rh == null || rh.tid != getThreadId(current))
                    rh = readHolds.get();
                int count = rh.count;
                if (count <= 1) {
                    readHolds.remove();
                    if (count <= 0)
                        throw unmatchedUnlockException();
                }
                --rh.count;
            }
            //CAS更新同步状态
            for (;;) {
                int c = getState();
                int nextc = c - SHARED_UNIT;
                if (compareAndSetState(c, nextc))
                    // Releasing the read lock has no effect on readers,
                    // but it may allow waiting writers to proceed if
                    // both read and write locks are now free.
                    return nextc == 0;
            }
        }
```

##### unmatchedUnlockException

```java
				//出现的情况是，unlock 读锁的线程，非获得读锁的线程。正常使用的情况，不会出现该情况。
        private IllegalMonitorStateException unmatchedUnlockException() {
            return new IllegalMonitorStateException(
                "attempt to unlock read lock, not locked by current thread");
        }
```

#### tryWriteLock

```java
        /**
         * Performs tryLock for write, enabling barging in both modes.
         * This is identical in effect to tryAcquire except for lack
         * of calls to writerShouldBlock.
         *
         * #tryWriteLock() 方法，尝试获取写锁。
         * 若获取成功，返回 true 。
         * 若失败，返回 false 即可，不进行等待排队。
         */
        final boolean tryWriteLock(){
            Thread current = Thread.currentThread();
            int c = getState();
            if(c != 0){
                // 获得现在写锁获取的数量
                int w = exclusiveCount(c);
                // 判断是否是其他的线程获取了写锁。若是，返回 false
                if(w == 0 || current != getExclusiveOwnerThread()){
                    return false;
                }
                // 超过写锁上限，抛出 Error 错误
                if(w == MAX_COUNT){
                    throw new Error("Maximum lock count exceeded");
                }
            }
            //  CAS 设置同步状态，尝试获取写锁。若失败，返回 false
            if(!compareAndSetState(c, c + 1)){
                return false;
            }
            // 设置持有写锁为当前线程
            setExclusiveOwnerThread(current);
            return true;
        }
```

#### tryReadLock

```java
        /**
         * Performs tryLock for read, enabling barging in both modes.
         * This is identical in effect to tryAcquireShared except for
         * lack of calls to readerShouldBlock.
         *
         * #tryReadLock() 方法，尝试获取读锁。
         * 若获取成功，返回 true 。
         * 若失败，返回 false 即可，不进行等待排队。
         */
        final boolean tryReadLock() {
            Thread current = Thread.currentThread();
            for (;;) {
                int c = getState();
                //exclusiveCount(c)计算写锁
                //如果存在写锁，且锁的持有者不是当前线程，直接返回-1
                //存在锁降级问题，后续阐述
                if (exclusiveCount(c) != 0 &&
                        getExclusiveOwnerThread() != current)
                    return false;
                // 读锁
                int r = sharedCount(c);
                if (r == MAX_COUNT)
                    throw new Error("Maximum lock count exceeded");
                if (compareAndSetState(c, c + SHARED_UNIT)) {
                    if (r == 0) {
                        firstReader = current;
                        firstReaderHoldCount = 1;
                    } else if (firstReader == current) {
                        firstReaderHoldCount++;
                    } else {
                        HoldCounter rh = cachedHoldCounter;
                        if (rh == null || rh.tid != getThreadId(current))
                            cachedHoldCounter = rh = readHolds.get();
                        else if (rh.count == 0)
                            readHolds.set(rh);
                        rh.count++;
                    }
                    return true;
                }
            }
        }
```

#### isHeldExclusively

```java
       protected final boolean isHeldExclusively() {
            // While we must in general read state before owner,
            // we don't need to do so to check if current thread is owner
            return getExclusiveOwnerThread() == Thread.currentThread();
        }
```

#### newCondition

```java
        // Methods relayed to outer class
        final ConditionObject newCondition() {
            return new ConditionObject();
        }
```

#### HoldCounter

```java
        /**
         * A counter for per-thread read hold counts.
         * Maintained as a ThreadLocal; cached in cachedHoldCounter
         *
         * 我们了解读锁的内在机制其实就是一个共享锁，为了更好理解 HoldCounter ，我们暂且认为它不是一个锁的概率，
         * 而相当于一个计数器。一次共享锁的操作就相当于在该计数器的操作。获取共享锁，则该计数器 + 1，释放共享锁，该计数器 - 1。
         * 只有当线程获取共享锁后才能对共享锁进行释放、重入操作。所以 HoldCounter 的作用就是当前线程持有共享锁的数量，
         * 这个数量必须要与线程绑定在一起，否则操作其他线程锁就会抛出异常。
         */
        static final class HoldCounter {
            int count = 0;// 计数器
            // Use id, not reference, to avoid garbage retention
            final long tid = getThreadId(Thread.currentThread());// 线程编号
        }

        /**
         * ThreadLocal subclass. Easiest to explicitly define for sake
         * of deserialization mechanics.
         *
         * 通过 ThreadLocalHoldCounter 类，HoldCounter 就可以与线程进行绑定了。故而，
         * HoldCounter 应该就是绑定线程上的一个计数器，而 ThreadLocalHoldCounter 则是线程绑定的 ThreadLocal。
         * 从上面我们可以看到 ThreadLocal 将 HoldCounter 绑定到当前线程上，同时 HoldCounter 也持有线程编号，
         * 这样在释放锁的时候才能知道 ReadWriteLock 里面缓存的上一个读取线程（cachedHoldCounter）是否是当前线程。
         * 这样做的好处是可以减少ThreadLocal.get() 方法的次调用数，因为这也是一个耗时操作。需要说明的是这样HoldCounter
         * 绑定线程编号而不绑定线程对象的原因是，避免 HoldCounter 和 ThreadLocal 互相绑定而导致 GC 难以释放它们
         * （尽管 GC 能够智能的发现这种引用而回收它们，但是这需要一定的代价），所以其实这样做只是为了帮助 GC 快速回收对象而已。
         */
        static final class ThreadLocalHoldCounter extends ThreadLocal<HoldCounter> {
            public HoldCounter initialValue() {
                return new HoldCounter();
            }
        }
```



### Sync 实现类

#### NonfairSync

```java
    /**
     * Nonfair version of Sync
     */
    static final class NonfairSync extends Sync {
        private static final long serialVersionUID = -8159625535654395037L;
        final boolean writerShouldBlock() {
            return false; // writers can always barge
        }
        //写锁是独占排它锁，所以在非公平锁的情况下，需要调用 AQS 的 			      #apparentlyFirstQueuedIsExclusive() 方法，判断是否当前写锁已经被获取。
        final boolean readerShouldBlock() {
            /* As a heuristic to avoid indefinite writer starvation,
             * block if the thread that momentarily appears to be head
             * of queue, if one exists, is a waiting writer.  This is
             * only a probabilistic effect since a new reader will not
             * block if there is a waiting writer behind other enabled
             * readers that have not yet drained from the queue.
             */
            return apparentlyFirstQueuedIsExclusive();
        }
    }
```

#### FairSync

```java
		/**
     * Fair version of Sync
     */
    static final class FairSync extends Sync {
        private static final long serialVersionUID = -2274990926593161451L;
        final boolean writerShouldBlock() {
            return hasQueuedPredecessors();
        }
        //调用 AQS 的 #hasQueuedPredecessors() 方法，是否有前序节点，即自己不是首个等待获取同步状态的节点。
        final boolean readerShouldBlock() {
            return hasQueuedPredecessors();
        }
    }
```


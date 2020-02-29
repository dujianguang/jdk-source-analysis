# LinkedBlockingDeque

## 1、介绍

BlockingQueue都是单向的FIFO队列，而LinkedBlockingDeque则是一个由链表组成的双向阻塞队列，双向队列就意味着可以从对头、对尾两端插入和移除元素，同样意味着LinkedBlockingDeque支持FIFO、FILO两种操作方式。

LinkedBlockingDeque是可选容量的，在初始化时可以设置容量防止其过度膨胀，如果不设置，默认容量大小为Integer.MAX_VALUE。

LinkedBlockingDeque 继承AbstractQueue，实现接口BlockingDeque，而BlockingDeque又继承接口BlockingQueue，BlockingDeque是支持两个附加操作的 Queue，这两个操作是：获取元素时等待双端队列变为非空；存储元素时等待双端队列中的空间变得可用。这两类操作就为LinkedBlockingDeque 的双向操作Queue提供了可能。

## 2、核心元素

### 2.1、重要参数

```java
    /**
     * Pointer to first node.
     * Invariant: (first == null && last == null) ||
     *            (first.prev == null && first.item != null)
     */
    transient Node<E> first;

    /**
     * Pointer to last node.
     * Invariant: (first == null && last == null) ||
     *            (last.next == null && last.item != null)
     */
    transient Node<E> last;

    /** Number of items in the deque */
    private transient int count;

    /** Maximum number of items in the deque */
    private final int capacity;

    /** Main lock guarding all access */
    final ReentrantLock lock = new ReentrantLock();

    /** Condition for waiting takes */
    private final Condition notEmpty = lock.newCondition();

    /** Condition for waiting puts */
    private final Condition notFull = lock.newCondition();
```

### 2.2、Node节点

```java
/** Doubly-linked list node class */
    static final class Node<E> {
        /**
         * The item, or null if this node has been removed.
         */
        E item;

        /**
         * One of:
         * - the real predecessor Node
         * - this Node, meaning the predecessor is tail
         * - null, meaning there is no predecessor
         */
        Node<E> prev;

        /**
         * One of:
         * - the real successor Node
         * - this Node, meaning the successor is head
         * - null, meaning there is no successor
         */
        Node<E> next;

        Node(E x) {
            item = x;
        }
    }
```

通过上面的Lock可以看出，LinkedBlockingDeque底层实现机制与LinkedBlockingQueue一样，依然是通过互斥锁ReentrantLock 来实现，notEmpty 、notFull 两个Condition做协调生产者、消费者问题。

## 3、核心方法

LinkedBlockingDeque 的add、put、offer、take、peek、poll系列方法都是通过调用XXXFirst，XXXLast方法。所以这里就仅以putFirst、putLast、pollFirst、pollLast分析下。

### 3.1、putFirst(E e)

```java
    /**
     * @throws NullPointerException {@inheritDoc}
     * @throws InterruptedException {@inheritDoc}
     *
     * 将指定的元素插入此双端队列的开头，必要时将一直等待可用空间。
     */
    public void putFirst(E e) throws InterruptedException {
        if (e == null) throw new NullPointerException();
        Node<E> node = new Node<E>(e);
        //获取锁
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            while (!linkFirst(node))
                // 在notFull条件上等待，直到被唤醒或中断
                notFull.await();
        } finally {
            lock.unlock();
        }
    }
```

先获取锁，然后调用linkFirst方法入列，最后释放锁。如果队列是满的则在notFull上面等待。linkFirst设置Node为对头：

#### linkFirst(Node node)

```java
/**
     * Links node as first element, or returns false if full.
     *
     * 设置node节点队列的列头节点，成功返回true，如果队列满了返回false。
     */
    private boolean linkFirst(Node<E> node) {
        // assert lock.isHeldByCurrentThread();
        // 超出容量
        if (count >= capacity)
            return false;
        // 新节点的next指向原first
        Node<E> f = first;
        node.next = f;
        // 设置node为新的first
        first = node;
        // 没有尾节点，设置node为尾节点
        if (last == null)
            last = node;
            // 有尾节点，那就将之前first的pre指向新增node
        else
            f.prev = node;
        ++count;
        // 唤醒notEmpty
        notEmpty.signal();
        return true;
    }
```



### 3.2、putLast(E e)

```java
/**
     * @throws NullPointerException {@inheritDoc}
     * @throws InterruptedException {@inheritDoc}
     *
     * 将指定的元素插入此双端队列的末尾，必要时将一直等待可用空间。
     */
    public void putLast(E e) throws InterruptedException {
        if (e == null) throw new NullPointerException();
        Node<E> node = new Node<E>(e);
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            //linkLast将节点Node链接到队列尾部
            while (!linkLast(node))
                notFull.await();
        } finally {
            lock.unlock();
        }
    }
```

#### linkLast(Node node)

```java
/**
     * Links node as last element, or returns false if full.
     *
     * 调用linkLast将节点Node链接到队列尾部
     */
    private boolean linkLast(Node<E> node) {
        // assert lock.isHeldByCurrentThread();
        if (count >= capacity)
            return false;
        // 尾节点
        Node<E> l = last;
        // 将Node的前驱指向原本的last
        node.prev = l;
        // 将node设置为last
        last = node;
        // 首节点为null，则设置node为first
        if (first == null)
            first = node;
        else
            //非null，说明之前的last有值，就将之前的last的next指向node
            l.next = node;
        ++count;
        notEmpty.signal();
        return true;
    }
```



### 3.3、pollFirst()

```java
//获取并移除此双端队列的第一个元素；如果此双端队列为空，则返回 null。
    public E pollFirst() {
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            //调用unlinkFirst移除队列首元素
            return unlinkFirst();
        } finally {
            lock.unlock();
        }
    }
```

#### unlinkFirst()

```java
    /**
     * Removes and returns first element, or null if empty.
     */
    private E unlinkFirst() {
        // assert lock.isHeldByCurrentThread();
        //首节点
        Node<E> f = first;
        //空队列,直接返回null
        if (f == null)
            return null;
        //first.next
        Node<E> n = f.next;
        //节点item
        E item = f.item;
        // 移除掉first ==> first = first.next
        f.item = null;
        f.next = f; // help GC
        first = n;
        // 移除后为空队列，仅有一个节点
        if (n == null)
            last = null;
        else
            // n的pre原来指向之前的first，现在n变为first了，pre指向null
            n.prev = null;
        --count;
        notFull.signal();
        return item;
    }
```



### 3.4、pollLast()

```java
//获取并移除此双端队列的最后一个元素；如果此双端队列为空，则返回 null。
    public E pollLast() {
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            //调用unlinkLast移除尾结点，链表空返回null
            return unlinkLast();
        } finally {
            lock.unlock();
        }
    }
```

#### unlinkLast()

```java
/**
     * Removes and returns last element, or null if empty.
     */
    private E unlinkLast() {
        // assert lock.isHeldByCurrentThread();
        Node<E> l = last;
        if (l == null)
            return null;
        Node<E> p = l.prev;
        E item = l.item;
        l.item = null;
        l.prev = l; // help GC
        last = p;
        if (p == null)
            first = null;
        else
            p.next = null;
        --count;
        notFull.signal();
        return item;
    }
```



LinkedBlockingDeque大部分方法都是通过linkFirst、linkLast、unlinkFirst、unlinkLast这四个方法来实现的，因为是双向队列，所以他们都是针对first、last的操作。
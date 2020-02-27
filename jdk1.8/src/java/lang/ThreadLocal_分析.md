# ThreadLocal

## 1、ThreadLocal是什么？

该类提供了线程局部 (thread-local) 变量。这些变量不同于它们的普通对应物，因为访问某个变量（通过其get 或 set方法）的每个线程都有自己的局部变量，它独立于变量的初始化副本。ThreadLocal实例通常是类中的 private static 字段，它们希望将状态与某一个线程（例如，用户 ID 或事务 ID）相关联。

所以ThreadLocal与线程同步机制不同，线程同步机制是多个线程共享同一个变量，而ThreadLocal是为每一个线程创建一个单独的变量副本，故而每个线程都可以独立地改变自己所拥有的变量副本，而不会影响其他线程所对应的副本。可以说ThreadLocal为多线程环境下变量问题提供了另外一种解决思路。

ThreadLocal定义了四个方法：

get()：返回此线程局部变量的当前线程副本中的值。

initialValue()：返回此线程局部变量的当前线程的“初始值”。

remove()：移除此线程局部变量当前线程的值。

set(T value)：将此线程局部变量的当前线程副本中的值设置为指定值。

除了这四个方法，ThreadLocal内部还有一个静态内部类ThreadLocalMap，该内部类才是实现线程隔离机制的关键，get()、set()、remove()都是基于该内部类操作。ThreadLocalMap提供了一种用键值对方式存储每一个线程的变量副本的方法，key为当前ThreadLocal对象，value则是对应线程的变量副本。

对于ThreadLocal需要注意的有两点：

ThreadLocal实例本身是不存储值，它只是提供了一个在当前线程中找到副本值得key。

是ThreadLocal包含在Thread中，而不是Thread包含在ThreadLocal中，有些小伙伴会弄错他们的关系。

Thread、ThreadLocal、ThreadLocalMap关系图

![thread_local_map](https://github.com/muyutingfeng/jdk-source-analysis/raw/master/note/doc/java.lang/ThreadLocal/thread_local_map%E5%85%B3%E7%B3%BB%E5%9B%BE.png?raw=true)

ThreadLocal的三个理论基础：

1、每个线程都有一个自己的ThreadLocal.ThreadLocalMap对象

2、每一个ThreadLocal对象都有一个循环计数器

3、ThreadLocal.get()取值，就是根据当前的线程，获取线程中自己的ThreadLocal.ThreadLocalMap，然后在这个Map中根据第二点中循环计数器取得一个特定value值



## 2、ThreadLocal使用示例

```java
package com.github.mf;

import java.util.Random;

public class ThreadLocalTest {
    private final static ThreadLocal<String> threadLocal = new ThreadLocal<>();
    private final static Random random = new Random(System.currentTimeMillis());

    public static void main(String[] args) throws InterruptedException {
        Thread t1 = new Thread(()->{
            threadLocal.set("Thread-1");
            try {
                Thread.sleep(random.nextInt(1000));
                System.out.println(Thread.currentThread().getName()+" "+threadLocal.get());
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        });

        Thread t2 = new Thread(()->{
            threadLocal.set("Thread-2");
            try {
                Thread.sleep(random.nextInt(1000));
                System.out.println(Thread.currentThread().getName()+" "+threadLocal.get());
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        });

        t1.start();
        t2.start();
        t1.join();
        t2.join();

        System.out.println("-------------------------------");
        System.out.println(Thread.currentThread().getName()+"  "+threadLocal.get());

    }
}
```



## 3、ThreadLocal源码解析

### ThreadLocalMap

ThreadLocalMap其内部利用Entry来实现key-value的存储，如下：

代码中可以看出Entry的key就是ThreadLocal，而value就是值。同时，Entry也继承WeakReference，所以说Entry所对应key（ThreadLocal实例）的引用为一个弱引用。

```java
				/**
         * The entries in this hash map extend WeakReference, using
         * its main ref field as the key (which is always a
         * ThreadLocal object).  Note that null keys (i.e. entry.get()
         * == null) mean that the key is no longer referenced, so the
         * entry can be expunged from table.  Such entries are referred to
         * as "stale entries" in the code that follows.
         */
        static class Entry extends WeakReference<ThreadLocal<?>> {
            /** The value associated with this ThreadLocal. */
            Object value;
            //内部利用Entry来实现key-value的存储
            Entry(ThreadLocal<?> k, Object v) {
                super(k);
                value = v;
            }
        }
```



### set方法(ThreadLocalMap)

```java
				/**
         * Set the value associated with key.
         *
         * @param key the thread local object
         * @param value the value to be set
         *  采用开放定址法
         *  replaceStaleEntry、cleanSomeSlots清除掉key==null的实例
         */
        private void set(ThreadLocal<?> key, Object value) {

            // We don't use a fast path as with get() because it is at
            // least as common to use set() to create new entries as
            // it is to replace existing ones, in which case, a fast
            // path would fail more often than not.

            Entry[] tab = table;
            int len = tab.length;
            //根据 ThreadLocal 的散列值，查找对应元素在数组中的位置
            int i = key.threadLocalHashCode & (len-1);
            //采用“线性探测法”，寻找合适位置
            for (Entry e = tab[i];
                 e != null;
                 e = tab[i = nextIndex(i, len)]) {
                ThreadLocal<?> k = e.get();
                // key 存在，直接覆盖
                if (k == key) {
                    e.value = value;
                    return;
                }
                // key == null，但是存在值（因为此处的e != null），说明之前的ThreadLocal对象已经被回收了
                if (k == null) {
                    // 用新元素替换陈旧的元素
                    replaceStaleEntry(key, value, i);
                    return;
                }
            }
            // ThreadLocal对应的key实例不存在也没有陈旧元素，new 一个
            tab[i] = new Entry(key, value);
            int sz = ++size;
            // cleanSomeSlots 清楚陈旧的Entry（key == null）
            // 如果没有清理陈旧的 Entry 并且数组中的元素大于了阈值，则进行 rehash
            if (!cleanSomeSlots(i, sz) && sz >= threshold)
                rehash();
        }
```

### threadLocalHashCode

```java
		/**
     * ThreadLocals rely on per-thread linear-probe hash maps attached
     * to each thread (Thread.threadLocals and
     * inheritableThreadLocals).  The ThreadLocal objects act as keys,
     * searched via threadLocalHashCode.  This is a custom hash code
     * (useful only within ThreadLocalMaps) that eliminates collisions
     * in the common case where consecutively constructed ThreadLocals
     * are used by the same threads, while remaining well-behaved in
     * less common cases.
     *
     * 定义为final，表示ThreadLocal一旦创建其散列值就已经确定了，生成过程则是调用nextHashCode()：
     */
    private final int threadLocalHashCode = nextHashCode();



    /**
     * Returns the next hash code.
     * nextHashCode表示分配下一个ThreadLocal实例的threadLocalHashCode的值，
     * HASH_INCREMENT则表示分配两个ThradLocal实例的threadLocalHashCode的增量，
     * 从nextHashCode就可以看出他们的定义。
     */
    private static int nextHashCode() {
        return nextHashCode.getAndAdd(HASH_INCREMENT);
    }
```



### getEntry方法

```java
/**
         * Get the entry associated with key.  This method
         * itself handles only the fast path: a direct hit of existing
         * key. It otherwise relays to getEntryAfterMiss.  This is
         * designed to maximize performance for direct hits, in part
         * by making this method readily inlinable.
         *
         * @param  key the thread local object
         * @return the entry associated with key, or null if no such
         *
         * 由于采用了开放定址法，所以当前key的散列值和元素在数组的索引并不是完全对应的，
         * 首先取一个探测数（key的散列值），如果所对应的key就是我们所要找的元素，则返回，
         * 否则调用getEntryAfterMiss()
         */
        private Entry getEntry(ThreadLocal key) {
            int i = key.threadLocalHashCode & (table.length - 1);
            Entry e = table[i];
            if (e != null && e.get() == key)
                return e;
            else
                return getEntryAfterMiss(key, i, e);
        }




				/**
         * Version of getEntry method for use when key is not found in
         * its direct hash slot.
         *
         * @param  key the thread local object
         * @param  i the table index for key's hash code
         * @param  e the entry at table[i]
         * @return the entry associated with key, or null if no such
         */
        private Entry getEntryAfterMiss(ThreadLocal key, int i, Entry e) {
            Entry[] tab = table;
            int len = tab.length;

            while (e != null) {
                ThreadLocal k = e.get();
                if (k == key)
                    return e;
                if (k == null)
                    //该方法用于处理key == null，有利于GC回收，能够有效地避免内存泄漏
                    expungeStaleEntry(i);
                else
                    i = nextIndex(i, len);
                e = tab[i];
            }
            return null;
        }
```

### *get方法*

```java
    /**
     * Returns the value in the current thread's copy of this
     * thread-local variable.  If the variable has no value for the
     * current thread, it is first initialized to the value returned
     * by an invocation of the {@link #initialValue} method.
     *
     * @return the current thread's value of this thread-local
     *
     * 返回当前线程所对应的线程变量
     */
    public T get() {
        //获取当前线程
        Thread t = Thread.currentThread();
        //获取当前线程的成员变量
        ThreadLocalMap map = getMap(t);
        if (map != null) {
            //从当前线程的ThreadLocalMap获取对应的Entry
            ThreadLocalMap.Entry e = map.getEntry(this);
            if (e != null) {
                @SuppressWarnings("unchecked")
                //获取目标值
                T result = (T)e.value;
                return result;
            }
        }
        return setInitialValue();
    }


		/**
     * Get the map associated with a ThreadLocal. Overridden in
     * InheritableThreadLocal.
     *
     * @param  t the current thread
     * @return the map
     * 获取当前线程所对应的ThreadLocalMap
     */
    ThreadLocalMap getMap(Thread t) {
        return t.threadLocals;
    }
```

### *set方法*

```java
		/**
     * Sets the current thread's copy of this thread-local variable
     * to the specified value.  Most subclasses will have no need to
     * override this method, relying solely on the {@link #initialValue}
     * method to set the values of thread-locals.
     *
     * @param value the value to be stored in the current thread's copy of
     *        this thread-local.
     *
     * 设置当前线程的局部变量的值
     */
    public void set(T value) {
        Thread t = Thread.currentThread();
        //获取当前线程所对应的ThreadLocalMap，
        ThreadLocalMap map = getMap(t);
        //如果不为空，则调用ThreadLocalMap的set()方法，key就是当前ThreadLocal
        //如果不存在，则调用createMap()方法新建一个
        if (map != null)
            map.set(this, value);
        else
            createMap(t, value);
    }



		/**
     * Create the map associated with a ThreadLocal. Overridden in
     * InheritableThreadLocal.
     *
     * @param t the current thread
     * @param firstValue value for the initial entry of the map
     */
    void createMap(Thread t, T firstValue) {
        t.threadLocals = new ThreadLocalMap(this, firstValue);
    }
```

### *initialValue方法*

```java
/**
     * Returns the current thread's "initial value" for this
     * thread-local variable.  This method will be invoked the first
     * time a thread accesses the variable with the {@link #get}
     * method, unless the thread previously invoked the {@link #set}
     * method, in which case the {@code initialValue} method will not
     * be invoked for the thread.  Normally, this method is invoked at
     * most once per thread, but it may be invoked again in case of
     * subsequent invocations of {@link #remove} followed by {@link #get}.
     *
     * <p>This implementation simply returns {@code null}; if the
     * programmer desires thread-local variables to have an initial
     * value other than {@code null}, {@code ThreadLocal} must be
     * subclassed, and this method overridden.  Typically, an
     * anonymous inner class will be used.
     *
     * @return the initial value for this thread-local
     * 返回该线程局部变量的初始值
     *
     * 该方法定义为protected级别且返回为null，很明显是要子类实现它的，
     * 所以我们在使用ThreadLocal的时候一般都应该覆盖该方法。
     * 该方法不能显示调用，只有在第一次调用get()或者set()方法时才会被执行，并且仅执行1次。
     */
    protected T initialValue() {
        return null;
    }
```

### *remove()方法*

```java
		/**
     * Removes the current thread's value for this thread-local
     * variable.  If this thread-local variable is subsequently
     * {@linkplain #get read} by the current thread, its value will be
     * reinitialized by invoking its {@link #initialValue} method,
     * unless its value is {@linkplain #set set} by the current thread
     * in the interim.  This may result in multiple invocations of the
     * {@code initialValue} method in the current thread.
     *
     * @since 1.5
     *
     * 将当前线程局部变量的值删除。该方法的目的是减少内存的占用。
     * 当然，我们不需要显示调用该方法，因为一个线程结束后，它所对应的局部变量就会被垃圾回收。
     */
     public void remove() {
         ThreadLocalMap m = getMap(Thread.currentThread());
         if (m != null)
             m.remove(this);
     }
```



## 4、ThreadLocal内存泄露

前面提到每个Thread都有一个ThreadLocal.ThreadLocalMap的map，该map的key为ThreadLocal实例，它为一个弱引用，我们知道弱引用有利于GC回收。当ThreadLocal的key == null时，GC就会回收这部分空间，但是value却不一定能够被回收，因为他还与Current Thread存在一个强引用关系。

![ThreadLocalGC](https://github.com/muyutingfeng/jdk-source-analysis/raw/master/note/doc/java.lang/ThreadLocal/ThreadLocalGC.png?raw=true)



由于存在这个强引用关系，会导致value无法回收。如果这个线程对象不会销毁那么这个强引用关系则会一直存在，就会出现内存泄漏情况。所以说只要这个线程对象能够及时被GC回收，就不会出现内存泄漏。如果碰到线程池，那就更坑了。

那么要怎么避免这个问题呢？

在前面提过，在ThreadLocalMap中的setEntry()、getEntry()，如果遇到key == null的情况，会对value设置为null。当然我们也可以显示调用ThreadLocal的remove()方法进行处理。

> ThreadLocal 不是用于解决共享变量的问题的，也不是为了协调线程同步而存在，而是为了方便每个线程处理自己的状态而引入的一个机制。这点至关重要。
>
> 每个Thread内部都有一个ThreadLocal.ThreadLocalMap类型的成员变量，该成员变量用来存储实际的ThreadLocal变量副本。
>
> ThreadLocal并不是为线程保存对象的副本，它仅仅只起到一个索引的作用。它的主要木得视为每一个线程隔离一个类的实例，这个实例的作用范围仅限于线程内部。



## 5、ThreadLocal原理

对这些内容做一个总结，ThreadLocal的原理简单说应该是这样的：

1. ThreadLocal不需要key，因为**线程里面自己的ThreadLocal.ThreadLocalMap不是通过链表法实现的，而是通过开放地址法实现的**
2. 每次set的时候往线程里面的ThreadLocal.ThreadLocalMap中的table数组某一个位置塞一个值，这个位置由ThreadLocal中的threadLocaltHashCode取模得到，如果位置上有数据了，就往后找一个没有数据的位置
3. 每次get的时候也一样，根据ThreadLocal中的threadLocalHashCode取模，取得线程中的ThreadLocal.ThreadLocalMap中的table的一个位置，看一下有没有数据，没有就往下一个位置找
4. 既然ThreadLocal没有key，那么一个ThreadLocal只能塞一种特定数据。如果想要往线程里面的ThreadLocal.ThreadLocalMap里的table不同位置塞数据 ，比方说想塞三种String、一个Integer、两个Double、一个Date，请定义多个ThreadLocal，ThreadLocal支持泛型"public class ThreadLocal<T>"。 
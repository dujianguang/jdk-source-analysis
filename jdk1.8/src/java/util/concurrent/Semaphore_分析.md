# Semaphore

信号量，Semaphore可以控同时访问的线程个数，通过 acquire() 获取一个许可，如果没有就等待，而 release() 释放一个许可。

<u>Semaphore是一个计数信号量，它的本质是一个"[共享锁](http://www.cnblogs.com/skywang12345/p/3505809.html)"。</u>

<u>信号量维护了一个信号量许可集。线程可以通过调用acquire()来获取信号量的许可；当信号量中有可用的许可时，线程能获取该许可；否则线程必须等待，直到有可用的许可为止。 线程可以通过release()来释放它所持有的信号量许可。</u>

### 1.0 semaphore的数据结构

![semaphore](/Users/muyutingfeng/Documents/company/github/jdk1.8-source-analysis/note/doc/java.util.concurrent.Semaphore/semaphore.jpg)

### 1.1、构造器

```java
/**
     * Creates a {@code Semaphore} with the given number of
     * permits and nonfair fairness setting.
     *
     * @param permits the initial number of permits available.
     *        This value may be negative, in which case releases
     *        must occur before any acquires will be granted.
     */
    public Semaphore(int permits) {
        sync = new NonfairSync(permits);
    }

    /**
     * Creates a {@code Semaphore} with the given number of
     * permits and the given fairness setting.
     *
     * @param permits the initial number of permits available.
     *        This value may be negative, in which case releases
     *        must occur before any acquires will be granted.
     * @param fair {@code true} if this semaphore will guarantee
     *        first-in first-out granting of permits under contention,
     *        else {@code false}
     * permits:表示许可数目，即同时可以允许多少线程进行访问
     * fair:表示是否是公平的，即等待时间越久的越先获取许可
     */
    public Semaphore(int permits, boolean fair) {
        sync = fair ? new FairSync(permits) : new NonfairSync(permits);
    }
```

### 1.2、核心方法

#### 1.2.1、acquire()、release()方法

```java
public void acquire() throws InterruptedException {  }     //获取一个许可
public void acquire(int permits) throws InterruptedException { }    //获取permits个许可
public void release() { }          //释放一个许可
public void release(int permits) { }    //释放permits个许可
```

acquire()用来获取一个许可，若无许可能够获得，则会一直等待，直到获得许可。

release()用来释放许可。注意，在释放许可之前，必须先获获得许可。

#### 1.2.2、tryAcquire方法

以上这4个方法都会被阻塞，如果想立即得到执行结果，可以使用下面几个方法：

```java
//尝试获取一个许可，若获取成功，则立即返回true，若获取失败，则立即返回false
public boolean tryAcquire() { };    
//尝试获取一个许可，若在指定的时间内获取成功，则立即返回true，否则则立即返回false
public boolean tryAcquire(long timeout, TimeUnit unit) throws InterruptedException { };  
//尝试获取permits个许可，若获取成功，则立即返回true，若获取失败，则立即返回false
public boolean tryAcquire(int permits) { }; 
//尝试获取permits个许可，若在指定的时间内获取成功，则立即返回true，否则则立即返回false
public boolean tryAcquire(int permits, long timeout, TimeUnit unit) throws InterruptedException { }; 

```

另外还可以通过availablePermits()方法得到可用的许可数目。

### 1.3、使用案例

假若一个工厂有5台机器，但是有8个工人，一台机器同时只能被一个工人使用，只有使用完了，其他工人才能继续使用。那么我们就可以通过Semaphore来实现：

```java
public class Test {
    public static void main(String[] args) {
        int N = 8;            //工人数
        Semaphore semaphore = new Semaphore(5); //机器数目
        for(int i=0;i<N;i++)
            new Worker(i,semaphore).start();
    }
     
    static class Worker extends Thread{
        private int num;
        private Semaphore semaphore;
        public Worker(int num,Semaphore semaphore){
            this.num = num;
            this.semaphore = semaphore;
        }
         
        @Override
        public void run() {
            try {
                semaphore.acquire();
                System.out.println("工人"+this.num+"占用一个机器在生产...");
                Thread.sleep(2000);
                System.out.println("工人"+this.num+"释放出机器");
                semaphore.release();           
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }
}

输出：
工人0占用一个机器在生产...
工人1占用一个机器在生产...
工人2占用一个机器在生产...
工人4占用一个机器在生产...
工人5占用一个机器在生产...
工人0释放出机器
工人2释放出机器
工人3占用一个机器在生产...
工人7占用一个机器在生产...
工人4释放出机器
工人5释放出机器
工人1释放出机器
工人6占用一个机器在生产...
工人3释放出机器
工人7释放出机器
工人6释放出机器
```

### 1.4、总结

下面对上面说的三个辅助类进行一个总结：

　　1）CountDownLatch和CyclicBarrier都能够实现线程之间的等待，只不过它们侧重点不同：

　　　　CountDownLatch一般用于某个线程A等待若干个其他线程执行完任务之后，它才执行；

　　　　而CyclicBarrier一般用于一组线程互相等待至某个状态，然后这一组线程再同时执行；

　　　　另外，CountDownLatch是不能够重用的，而CyclicBarrier是可以重用的。

　　2）Semaphore其实和锁有点类似，它一般用于控制对某组资源的访问权限。


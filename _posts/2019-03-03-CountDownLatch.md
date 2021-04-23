---
layout: post
title:  "CountDownLatch-Semaphore"
categories: thread
tags:  thread
author: 网络

---

* content
  {:toc}

总结java线程CountDownLatch基础知识

## CountDownLatch

### CountDownLatch是什么?

CountDownLatch这个类能够使一个线程等待其他线程完成各自的工作后再执行。例如，应用程序的主线程希望在负责启动框架服务的线程已经启动所有的框架服务之后再执行。主线程必须在启动其他线程后立即调用CountDownLatch.await()方法。这样主线程的操作就会在这个方法上阻塞，直到其他线程完成各自的任务。

```java
//伪代码
//Main thread start
//Create CountDownLatch for N threads
//Create and start N threads
//Main thread wait on latch
//N threads completes there tasks are returns
//Main thread resume execution
```

### 使用场景

* 实现最大的并行性

有时我们想同时启动多个线程，实现最大程度的并行性。例如，我们想测试一个单例类。如果我们创建一个初始计数为1的CountDownLatch，并让所有线程都在这个锁上等待，那么我们可以很轻松地完成测试。我们只需调用 一次countDown()方法就可以让所有的等待线程同时恢复执行。

* 开始执行前等待n个线程完成各自任务，见下方示例代码

### 示例

应用程序启动前检查各个外部服务的状态

```java
//入口程序
public class MyApplication {
    public static void main(String[] args) {
        boolean result = false;
        try {
            result = ApplicationStartupUtil.checkExternalServices();
        } catch (Exception e) {
            e.printStackTrace();
        }
        System.out.println("External services validation completed !! Result was :: "+ result);
    }
}

//服务检查工具类
public class ApplicationStartupUtil
{
    //List of service checkers
    private static List<BaseHealthChecker> _services;

    //This latch will be used to wait on
    private static CountDownLatch _latch;

    private ApplicationStartupUtil()
    {
    }

    private final static ApplicationStartupUtil INSTANCE = new ApplicationStartupUtil();

    public static ApplicationStartupUtil getInstance()
    {
        return INSTANCE;
    }

    public static boolean checkExternalServices() throws Exception
    {
        //Initialize the latch with number of service checkers
        _latch = new CountDownLatch(3);

        //All add checker in lists
        _services = new ArrayList<BaseHealthChecker>();
        _services.add(new NetworkHealthChecker(_latch));
        _services.add(new CacheHealthChecker(_latch));
        _services.add(new DatabaseHealthChecker(_latch));

        //Start service checkers using executor framework
        Executor executor = Executors.newFixedThreadPool(_services.size());

        for(final BaseHealthChecker v : _services)
        {
            executor.execute(v);
        }

        //Now wait till all services are checked
        _latch.await();

        //Services are file and now proceed startup
        for(final BaseHealthChecker v : _services)
        {
            if( ! v.isServiceUp())
            {
                return false;
            }
        }
        return true;
    }
}

//服务检查的基类
public abstract class BaseHealthChecker implements Runnable {

    private CountDownLatch _latch;

    private String _serviceName;
    private boolean _serviceUp;

    //Get latch object in constructor so that after completing the task, thread can countDown() the latch
    public BaseHealthChecker(String serviceName, CountDownLatch latch)
    {
        super();
        this._latch = latch;
        this._serviceName = serviceName;
        this._serviceUp = false;
    }

    @Override
    public void run() {
        try {
            verifyService();
            _serviceUp = true;
        } catch (Throwable t) {
            t.printStackTrace(System.err);
            _serviceUp = false;
        } finally {
            if(_latch != null) {
                _latch.countDown();
            }
        }
    }

    public String getServiceName() {
        return _serviceName;
    }

    public boolean isServiceUp() {
        return _serviceUp;
    }
    //This methos needs to be implemented by all specific service checker
    public abstract void verifyService();
}

//检查网络的实现类
public class NetworkHealthChecker extends BaseHealthChecker
{
    public NetworkHealthChecker (CountDownLatch latch)  {
        super("Network Service", latch);
    }

    @Override
    public void verifyService()
    {
        System.out.println("Checking " + this.getServiceName());
        try
        {
            Thread.sleep(7000);
        }
        catch (InterruptedException e)
        {
            e.printStackTrace();
        }
        System.out.println(this.getServiceName() + " is UP");
    }
}
//检查缓存的实现类
public class CacheHealthChecker extends BaseHealthChecker{//实现同上}
//检查数据库的实现类
public class DatabaseHealthChecker extends BaseHealthChecker{//实现同上}
```

### 原理

```java
public class CountDownLatch {
    //利用了AQS内部类的共享锁的机制来实现的
    private static final class Sync extends AbstractQueuedSynchronizer {
        private static final long serialVersionUID = 4982264981922014374L;

        Sync(int count) {
            setState(count);
        }

        int getCount() {
            return getState();
        }

        protected int tryAcquireShared(int acquires) {
            //调用await的时候其实就是判断state是否被减到0了
            return (getState() == 0) ? 1 : -1;
        }

        protected boolean tryReleaseShared(int releases) {
            // Decrement count; signal when transition to zero
            for (;;) {
                int c = getState();
                if (c == 0)
                    return false;
                int nextc = c-1;
                if (compareAndSetState(c, nextc))
                    return nextc == 0;
            }
        }
    }

    private final Sync sync;

    //1. 初始化的时候直接设置了state为一个数值
    public CountDownLatch(int count) {
        if (count < 0) throw new IllegalArgumentException("count < 0");
        this.sync = new Sync(count);
    }

    public void await() throws InterruptedException {
        //2. 调用了内部类的Sync.tryAcquireShared()方法
        //其实就是判断state是否等于0了
        sync.acquireSharedInterruptibly(1);
    }

    public boolean await(long timeout, TimeUnit unit)
        throws InterruptedException {
        return sync.tryAcquireSharedNanos(1, unit.toNanos(timeout));
    }

    //3. 将state数值减1
    public void countDown() {
        sync.releaseShared(1);
    }

    public long getCount() {
        return sync.getCount();
    }

    public String toString() {
        return super.toString() + "[Count = " + sync.getCount() + "]";
    }
}
```

## Samaphore

### Samaphore是什么？

不可重入，支持公平信号量、非公平信号量，默认非公平

可以用于资源访问限流，连接池，缓存池等

Semaphore只是控制线程的数量，并不能实现同步，所以如果需要同步还是加锁或使用同步机制。信号量只是在信号不够的时候挂起线程，但是并不能保证信号量足够的时候获取对象和返还对象是线程安全的。

非公平信号量的吞吐量总是要比公平信号量的吞吐量要大，但是需要强调的是非公平信号量和非公平锁一样存在“饥渴死”的现象，也就是说活跃线程可能总是拿到信号量，而非活跃线程可能难以拿到信号量。而对于公平信号量由于总是靠请求的线程的顺序来获取信号量，所以不存在此问题。

### 示例

```java
final Semaphore semaphore = new Semaphore(2);
        ExecutorService executorService = Executors.newCachedThreadPool();
        for (int i = 0; i < 10; i++) {
            final int index = i;
            executorService.execute(new Runnable() {
                @Override
                public void run() {
                    try {
                        semaphore.acquire();
                        System.out.println("线程:" + Thread.currentThread().getName() + "获得许可:" + index);
                        TimeUnit.SECONDS.sleep(1);
                        semaphore.release();
                        System.out.println("TASK个数：" + semaphore.availablePermits());
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
            });
        }
        executorService.shutdown();
```

## CyclicBarrier

### 什么是CyclicBarrier?

CyclicBarrier翻译过来也叫栅栏，意思很明显，就是一组线程相互等待，均到达栅栏的时候，再运行。CyclicBarrier是可以重复使用的，而之前的CountDownLatch是一次性的。CyclicBarrier允许一组线程相互等待，直到到达某个公共屏障点，屏障点即一组任务执行完毕的时候。

### 示例

```java
import java.util.concurrent.BrokenBarrierException;
import java.util.concurrent.CountDownLatch;
import java.util.concurrent.CyclicBarrier;

public class RunningMan extends Thread{

    private String name;
    private long speed;
    private CyclicBarrier cyclicBarrier;
    private CountDownLatch countDownLatch;
    public RunningMan(String name,long speed,CyclicBarrier cyclicBarrier,CountDownLatch countDownLatch){
        this.name=name;
        this.speed=speed;
        this.cyclicBarrier=cyclicBarrier;
        this.countDownLatch=countDownLatch;
    }

    @Override
    public void run(){
        running();
        celebrate();
        sayGoodBy();
    }
  
    private void running(){ 
        System.out.println("Waiting "+name);
        try {
            //所有人开始等待人到期，最后一个人到期后就开始跑了
            int index=cyclicBarrier.await();
            System.out.println("index is "+index);
        } catch (InterruptedException e) {
            e.printStackTrace();
        } catch (BrokenBarrierException e) {
            e.printStackTrace();
        }
        System.out.println("Running "+speed);
        try {
            Thread.sleep(speed);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }

    private void celebrate(){
        System.out.println("Celebrate man comming "+name);
        try {
            int index=cyclicBarrier.await();  //等待人到场后开始庆祝喝酒
            System.out.println("index is "+index);
        } catch (InterruptedException e) {
            e.printStackTrace();
        } catch (BrokenBarrierException e) {
            e.printStackTrace();
        }
        System.out.println("Drinking "+speed);
        try {
            Thread.sleep(speed);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }

    private void sayGoodBy(){
        try {
            System.out.println(name+":say good bye");
            //等待每个人say good bye
            int index=cyclicBarrier.await();
            //CountDownLatch减1
            countDownLatch.countDown();
        } catch (InterruptedException e) {
            e.printStackTrace();
        } catch (BrokenBarrierException e) {
            e.printStackTrace();
        }
    }
}
```

```java
public class CyclicBarrierTest {
    // 定义一个CyclicBarrier，可以重复使用
    private static CyclicBarrier cyclicBarrier=new CyclicBarrier(5);
    // 定义一个CountDownLatch，用于最后每个人say bye后打印，bye bye
    private static CountDownLatch countDownLatch=new CountDownLatch(5);

    public static void main(String[] args){

        for(int i=1;i<=5;i++){

            new RunningMan("name"+i,Long.valueOf(i*1000),cyclicBarrier,countDownLatch).start();
        }
        try {
            countDownLatch.await();  //主线程等待最后bye bye
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println("ByeBye");

    }
}
```

### 与CountDownLatch的区别

1. CountDownLatch是一次性的，不可重复使用，CyclicBarrier可以重复使用
2. CountDownLatch一般用于某个线程A等待若干个其他线程执行完任务之后它才执行（递减计数器），CyclicBarrier一般用于一组线程互相等待至某个状态，然后这一组线程再同时执行（递增计数器）


## 参考

[什么时候使用CountDownLatch](http://www.importnew.com/15731.html)

[Semaphore](https://www.cnblogs.com/dpains/p/7526923.html)

[CyclicBarrier](https://www.cnblogs.com/dpains/p/7525397.html)

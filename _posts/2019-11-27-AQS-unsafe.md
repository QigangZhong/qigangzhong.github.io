---
layout: post
title:  "AQS、unsafe"
categories: thread
tags:  thread
author: 网络
---

* content
{:toc}

总结java线程基础知识

* aqs、unsafe









## 一. unsafe

park & unpark & LockSupport

```java
public class UnsafeUtil {
    private static final Unsafe THE_UNSAFE;
    static{
        try {
            final PrivilegedExceptionAction<Unsafe> action = () -> {
                Field theUnsafe = Unsafe.class.getDeclaredField("theUnsafe");
                theUnsafe.setAccessible(true);
                return (Unsafe) theUnsafe.get(null);
            };
            THE_UNSAFE = AccessController.doPrivileged(action);
        }
        catch (Exception e){
            throw new RuntimeException("Unable to load unsafe", e);
        }
    }

    public static Unsafe getUnsafe() {
        return THE_UNSAFE;
    }
}
```

```java
/**
 * 1. park:
 * public native void park(boolean isAbsolute, long time);
 * 调用park后，线程将一直阻塞直到超时或者中断等条件出现
 * 2. unpark:
 * unpark可以终止一个挂起的线程，使其恢复正常。整个并发框架中对线程的挂起操作被封装在 LockSupport类中，LockSupport类中有各种版本pack方法，但最终都调用了Unsafe.park()方法
 * unpark方法最好不要在调用park前对当前线程调用unpark
 */
public class Park_Unpark {
    public static void main(String[] args) {
        //【1】测试park方法阻塞线程，超时后自动恢复执行
        /*Unsafe unsafe = UnsafeUtil.getUnsafe();
        //超时时间为0，当前线程一直阻塞
        //unsafe.park(false, 0);
        //设置相对超时时间，相对时间后面的参数单位是纳秒
        //unsafe.park(false, 3000000000L);
        //设置绝对时间，绝对时间后面的参数单位是毫秒
        long time = System.currentTimeMillis()+3000;
        unsafe.park(true, time);
        System.out.println("SUCCESS!!!");*/


        //【2】测试park方法阻塞线程，被interrupt后，或者被unpark后自动恢复执行
        Unsafe unsafe = UnsafeUtil.getUnsafe();
        Thread currThread = Thread.currentThread();
        new Thread(()->{
            try {
                Thread.sleep(3000);
                currThread.interrupt();
                //unsafe.unpark(currThread);
            } catch (Exception e) {}
        }).start();
        unsafe.park(false, 0);
        System.out.println("SUCCESS!!!");
    }
}
```

## 二、ReentrantLock、ReentrantReadWriteLock、CountDownLatch、CyclicBarrier、Semaphor

## 参考

[Java并发之AQS详解](https://www.cnblogs.com/waterystone/p/4920797.html)

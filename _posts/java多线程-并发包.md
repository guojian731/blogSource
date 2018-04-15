---
title: java多线程-并发包
date: 2017-12-26 19:37:41
tags: java多线程
---
**因为本人的知识有限，并发包里我指挥记录一些我个人理解相对比较深的，之后会在慢慢的学习和实践中继续更近我的博客。**

# 重入锁

重入锁完全可以代替synchronized关键字。在jdk1.5以前，重入锁的性能远大于synchronized，但是在jdk1.6之后，两者的性能差别不大。

## 核心方法

lock.lock():进行上锁操作;
lock.unclock():解锁操作;
lock.lockInterruptibly()：获得锁，但是优先相应中断。
lock.tryLock(): 判断当前锁是否被占用，如果没有被占用，返回true，并且占有锁。如果被占用，返回false，那么直接退出。
lock.tryLock(Long time,TimeUnit unit):time是代表时间，unit代表单位，在设置的给定时间内获得锁。
lock.isHeldByCurrentThread():判断当前线程是否占有锁，true是占有。
lock.interrupt():锁中断；



## 简单的例子

重入锁的使用java.util.concurrent.ReentrantLock类来实现。下面是一个重入锁的例子。

``` java

import java.util.concurrent.locks.ReentrantLock;

public class Test  implements Runnable{
    private  int i=0;
    private static ReentrantLock lock =new ReentrantLock();
    @Override
    public  void run() {
        for (int k=0;k<10000;k++){
            lock.lock();
            try{
                i++;
            }finally {
                lock.unlock();
            }
        }
    }

    public static void main(String[] args) throws InterruptedException {
        Test test=new Test();
        Thread t1=new Thread(test,"t1");
        Thread t2=new Thread(test,"t2");
        t1.start();
        t2.start();
        t1.join();
        t2.join();
        System.out.println(test.i);
    }
}

```

这里使用lock锁住临界区的资源i，确保多线程对i进行操作的安全性，他还有一个好处，就是synchronized是修饰不了int的，只能修饰Integer类型，因为synchronized只能修饰对象，而int不是对象，是基本类型。

为什么ReentrantLock叫做重入锁呢，因为一个锁可以锁对象多次。但是锁住锁和释放锁的数量应该是一样的，只加锁而不释放锁，会让线程一直占有锁，从而使其他线程一直阻塞。

``` java

import java.util.concurrent.locks.ReentrantLock;

public class Test  implements Runnable{
    private  int i=0;
    private static ReentrantLock lock =new ReentrantLock();
    @Override
    public  void run() {
        for (int k=0;k<10000;k++){ 
            lock.lock();
            lock.lock();
            try {
                i++;
            }finally {
                lock.unlock();
                lock.unlock();
            }
        }
    }

    public static void main(String[] args) throws InterruptedException {
        Test test=new Test();
        Thread t1=new Thread(test,"t1");
        Thread t2=new Thread(test,"t2");
        t1.start();
        t2.start();
        t1.join();
        t2.join();
        System.out.println(test.i);
    }
}

```

## 中断响应

** 我们再使用synchronized的时候，如果一个线程在等待锁，要不一直在那等待，要不获得锁继续执行，没有第二种可能。但是如果使用重入锁，那么你将会有第三种可能，就是根据需求取消对锁的请求。如果一个线程正在等待锁，那么他依然可能收到一个通知，被告知无须等待，可以停止工作了，这样就减少了死锁。**

``` java

import java.util.concurrent.locks.ReentrantLock;

public class LockTest  implements Runnable{

    public static ReentrantLock lock1=new ReentrantLock();
    public static ReentrantLock lock2=new ReentrantLock();
    int lock;

    public LockTest(int lock){
        this.lock=lock;
    }
    @Override
    public void run() {
        try {
            if (lock==1){
                lock1.lockInterruptibly();
                System.out.println(Thread.currentThread().getName()+" 持有了锁lock1");
                Thread.sleep(10000);

                lock2.lockInterruptibly();
                System.out.println(Thread.currentThread().getName()+" 持有了锁lock2");
            }else {
                lock2.lockInterruptibly();
                System.out.println(Thread.currentThread().getName()+" 持有了锁lock2");
                try {
                    Thread.sleep(10000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                lock1.lockInterruptibly();
                System.out.println(Thread.currentThread().getName()+" 持有了锁lock1");
            }
        } catch (InterruptedException e) {
            e.printStackTrace();
        }finally {
            if (lock1.isHeldByCurrentThread()){
                lock1.unlock();
            }
            if (lock2.isHeldByCurrentThread()){
                lock2.unlock();
            }
        }
    }

    public static void main(String[] args) {
        LockTest test1=new LockTest(1);
        LockTest test2=new LockTest(2);
        Thread t1=new Thread(test1,"t1");
        Thread t2=new Thread(test2,"t2");
        t1.start();
        t2.start();
        try {
            Thread.sleep(10000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        t2.interrupt();
    }

}

```

在这个例子中，我们一个类有两个锁，线程t1先持有lock1，在持有lock2。而线程t2持有lock2，在持有lock1，因为我们使用了Thread.sleep(),确定了t1获得了lock1的锁，而t2获得了lock2的锁，这样就会产生死锁，在线程运行的十秒钟里(Thread.sleep(10000))，一直没有人释放资源，就是两个线程就会一直锁死，当主线程出来解决矛盾，让t2取消请求，t2就会释放锁lock2，这样t1就会获得了锁lock2。

![jvm装载步骤](java多线程-并发包/bf1.png)

## 给请求设置时间

**我们可以用tryLock()方法来给请求设置时间，它有两个参数，一个是数值，另一个是时间单位。lock.tryLock(5, TimeUnit.SECONDS)就是如果请求如果等待了5秒钟以上，就退出。如果为设置参数，那么就是请求的时候有其他线程持有锁，就退出。如果没有线程持有锁，就执行。**

``` java

import java.util.concurrent.TimeUnit;
import java.util.concurrent.locks.ReentrantLock;

public class ThreadTimeExample implements Runnable{

    public static ReentrantLock lock=new ReentrantLock();

    @Override
    public void run() {
        try {
            if (lock.tryLock(5, TimeUnit.SECONDS)) {
                Thread.sleep(6000);
                System.out.println(Thread.currentThread().getName()+"执行了");
            }else {
                System.out.println(Thread.currentThread().getName()+"未执行");
            }
        } catch (InterruptedException e) {
            e.printStackTrace();
        }finally {
            if (lock.isHeldByCurrentThread())
            lock.unlock();
        }
    }

    public static void main(String[] args) {
        ThreadTimeExample t=new ThreadTimeExample();
        Thread t1=new Thread(t,"t1");
        Thread t2=new Thread(t,"t2");
        Thread t3=new Thread(t,"t3");
        t1.start();
        t2.start();
        try {
            Thread.sleep(2000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        t3.start();
    }
}

```


我们新建了三个线程t1，t2,t3三个线程，因为t1,t2几乎同时启动，假设t1先持有锁(因为t1，t2都有机会先获得锁)，持有了锁6s，超过了t2等待时间5s，所以t2就不会运行而是选择退出。当t1运行2s中之后，t3才开始请求锁，相对于t3而言，t1持有锁的时间为大约为4s，没超过5s，所以t3也会执行的。

![jvm装载步骤](java多线程-并发包/bf2.png)

我们要注意一下finally的方法，当我们把finally的方法去掉，看一下运行情况。

![jvm装载步骤](java多线程-并发包/bf3.png)

发现只有t1线程获得了锁，其他线程都没获得到锁，t2没有获得锁正常，是在我们的预想之内，为什么t3也没获得锁呢，这是不应该的。那是因为t1在获得锁之后，没有执行lock.unlock()来释放锁，所以它持有锁的时间并不是6s而是更长。
好了，我们知道原因了，我们在finally代码快中添加lock.unlock()方法，运行看一下结果把。

![jvm装载步骤](java多线程-并发包/bf4.png)

我们发现竟然报错了，为什么呢？其实真正的原因就是线程t2，t2根本没有获得过锁，我们执行lock.unlock()当然报错了，之前我们就说过，锁的持有和释放必须是等对的，t2没有获得锁，却要释放锁，当然要报错了。所以我们要在释放锁之前做一下判断当前线程是否获得过锁 （lock.isHeldByCurrentThread())，如果获得过，释放。

## lock设置公平锁

在锁的竞争中，锁被占用并不是根据来的先后顺序来进行获得锁的，就会出现后来者居上的情况，如何对获得锁需要按照先后顺序来呢，synchronized就无法实现了，但是ReenTrantLock可以。
设置起来也很简单，就是在新建Lock对象的是够，传参true。

``` java
import java.util.concurrent.locks.ReentrantLock;

public class FairReenTrantLock implements Runnable{
    private static ReentrantLock lock=new ReentrantLock(true);
    @Override
    public void run() {

        for (int i=0;i<10;i++){
            lock.lock();
            System.out.print(Thread.currentThread().getName()+" ");
            lock.unlock();
        }

    }
    public static void main(String[] args) {
        FairReenTrantLock lock=new FairReenTrantLock();
        Thread t1=new Thread(lock,"t1");
        Thread t2=new Thread(lock,"t2");
        Thread t3=new Thread(lock,"t3");
        Thread t4=new Thread(lock,"t4");
        t1.start();
        t2.start();
        t3.start();
        t4.start();

    }
}
```

![jvm装载步骤](java多线程-并发包/bf5.png)

我们看一下结果，因为最开始的时候，有线程还处于未启动状态，当线程全部启动后，他们获取锁是有顺序的，最先得到锁的线程释放锁之后，就或在申请锁，所以轮一圈后，它依旧先获得锁。

# Condition条件

它的作用与Object.wait()和Object.notify()，Object.wait()和Object.notify()必须在synchronized修饰的方法块里才能使用，同样的Condition需要与重入锁相关联，在lock.newCondition()来为重入锁绑定一个Condition实例。

``` java

import java.util.concurrent.locks.Condition;
import java.util.concurrent.locks.ReentrantLock;

public class ReentranctLockConditine implements Runnable{

    public static ReentrantLock lock =new ReentrantLock();
    public static Condition condition=lock.newCondition();

    @Override
    public void run() {
        lock.lock();
        try {
            condition.await();
            System.out.println(Thread.currentThread().getName()+"线程继续运行");
        } catch (InterruptedException e) {
            e.printStackTrace();
        }finally {
            lock.unlock();
        }
    }

    public static void main(String[] args) {
        try {
            ReentranctLockConditine reentranctLockConditine=new ReentranctLockConditine();
            Thread t=new Thread(reentranctLockConditine,"t1");
            t.start();
            Thread.sleep(2000);
            lock.lock();
            condition.signalAll();
            lock.unlock();
            System.out.println("主线程结束了");
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}

```

首先我们在main()方法中建立一个线程并启动，线程获得锁后，执行 condition.await();，要求condition在这个对象上进行等待，这时候线程会释放锁和资源，当condition.signalAll();会唤醒所有的资源，之前的锁又会尝试绑定之前的锁，这期间，需要获得锁的线程(也就是现在的主线程main方法)释放锁，要不然线程t1就会只与锁进行绑定，而不会获得锁。如果你讲33行的代码注掉的话，你会发现t1是无法执行到输入的语句的。


















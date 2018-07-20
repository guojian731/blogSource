---
title: java多线程-wait、notify、notifyAll
date: 2018-07-16 19:53:45
tags: java多线程
---

### wait、notify、notifyAll难点 ###

***前提首先是需要有共享变量，这个变量如果不满足条件，该线程执行wait()就会处于等待状态，但是这个变量不会永远处于不满足状态，当共享变量满足条件后，再将线程唤醒。wait()的作用就是将线程进行等待，notify()的作用就是唤醒一个被暂停的线程。***

***很显然，提到了共享变量，所以这个过程是原子性的，必须在synchronize修饰过的方法或者代码块中才能使用wait()。（也就是说，调用一个对象的wait方法，必须获得这个对象的内部锁）***

**在Object对象中，维护了一个因为该对象暂停的线程队列，该对象wait()的时候就会往队列里添加一个，notify()的时候就会唤醒其中一个线程。所以wait()、notify()在Object类里能更好的管理。**

**wait()方法会释放该对象的内部锁，试想一下，别人线程需要更改该对象(共享变量)的状态，首先需要获取该对象的锁，如果wait()方法不释放锁，其他线程又怎么能获得锁呢。（wait()方法暂停当前线程释放的锁只是wait对象所属对象的内部锁，当前线程持有的其他锁、显示锁并不会因此而释放）。**

wait()方法并不是简单的释放锁、暂停线程就结束了。而是包括加入等待集，暂停当前线程、释放锁以及将唤醒后的等待线程从等待集中移除等，都是在Object.wait()中实现的。wait()伪代码：

``` java 
public void wait(){
	atomic{//原子操作开始
	//释放当前内部锁
	releaseLock(this);
	//暂停当前线程
	block(Thread.currentThread());//语句1
	}//原子操作结束

	//再次申请当前对象的内部锁
	acquireLock(this);//语句2
	//将当前线程从当前对象的等待中移除
	removeFromWaitSet(Thread.currentThread());
	return;//返回
}
```

***等待线程在语句1执行的时候被暂停了。被唤醒的线程在其占用处理器继续运行的时候会继续执行其他暂停前调用的Object.wait()中的其他指令，即从上述代码中的语句2开始继续执行：先再次申请Object.wait()所属对象的内部锁，接着将当前线程从相应的等待线程中移除，然后Object.wait()调用才返回。***

***Object.wait()的执行线程会一直处于Waiting状态，知道通知线程唤醒该线程并且保护条件成立。因此，Object.wait()所实现的等待是无限等待，而Object.wait(long)允许我们指定一个超时时间，如果被暂停的等待线程在这个时间内没有被其他线程唤醒，那么Java虚拟机会自动唤醒该线程。不过Object.wait(long)即无返回值也不会抛出特定的异常，以便区分其返回是由于其他线程通知了当前线程还是由于等待超时。因为，使用Object.wai(long)的时候我们需要一些额外的处理。***

### wait()场景题 ### 

1.现在有一个大的水桶100L，每三秒会有机器将水桶注满;有三个管道可以输出水，每个人都可以申请水0~100L,如果申请的水资源不够或者没有空余的管道，就需要等待。

Bucket，水桶类

``` java 

public class Bucket{
    
    private volatile AtomicInteger capacity=new AtomicInteger(100);//容量
    
    private volatile AtomicInteger length=new AtomicInteger(3);//管道数
    
    /**
     * 取水方法
     * @param count
     * @throws InterruptedException
     */
    public void takeWater(int count) throws InterruptedException {
        synchronized (this){
            while (count>capacity.get() || length.get()<=0){
                System.out.println("无法获取水，取水数量："+count+"；当前容量："+capacity+"；当前管道数："+length);
                try {
                    this.wait();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
            System.out.println("开始取水："+count+"；当前容量："+capacity+"；当前管道数："+length);
            length.decrementAndGet();//取水占用管道
            capacity.addAndGet(-count);//取水占用容量
            this.notify();
        }
        Thread.sleep(1000);//模拟取水的过程
        length.incrementAndGet();//取完水之后放开管道
        System.out.println("取水完成："+count+";当前容量："+capacity+";当前管道数："+length);
        
    }
    
    
    public void inWater(){
        synchronized (this){
            capacity.set(100);
            System.out.println("将水库的水填满");
            this.notifyAll();
        }
    }
    
}

```

取水的线程。
``` java 

public class TakeWater implements Runnable{
    private Bucket bucket;
    private int count;
    public TakeWater(Bucket reservoir,int count){
        this.bucket=reservoir;
        this.count=count;
    }
    @Override
    public void run() {
        try {
            bucket.takeWater(count);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
    
}

```

注水的线程

``` java 

public class InWater implements Runnable{
    private Bucket bucket;
    public InWater(Bucket reservoir){
        this.bucket=reservoir;
    }
    @Override
    public void run() {
        while (true){
            try {
                Thread.sleep(3000);
                bucket.inWater();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }
}

```
场景模拟：
``` java 
public class Client {
    
    public static void main(String[] args) throws InterruptedException {
        Bucket bucket=new Bucket();
        Integer[] table={30,70,80,100};
        for (int i=0;i<table.length;i++){
            Thread t2=new Thread(new TakeWater(bucket,table[i]));
            t2.start();
        }
        Thread t3=new Thread(new InWater(bucket));
        t3.start();
    }
}

```

运行结果

``` java
开始取水：30；当前容量：100；当前管道数：3
开始取水：70；当前容量：70；当前管道数：2
无法获取水，取水数量：80；当前容量：0；当前管道数：1
无法获取水，取水数量：100；当前容量：0；当前管道数：1
取水完成：70;当前容量：0当前管道数;3
取水完成：30;当前容量：0当前管道数;3
将水库的水填满
开始取水：100；当前容量：100；当前管道数：3
无法获取水，取水数量：80；当前容量：0；当前管道数：2
取水完成：100;当前容量：0当前管道数;3
将水库的水填满
开始取水：80；当前容量：100；当前管道数：3
取水完成：80;当前容量：20当前管道数;3
将水库的水填满
将水库的水填满
...
```
### wait/notify的开销问题 ###

- 过早唤醒问题：现在有一个房地产开发商发放礼物，总共有十个柜台，如果柜台有空余的话，买房的人(vip)和没买房的人(normal)都可以领取礼物，当时一旦发放礼物的柜台都满的情况下，当空出柜台的时候，优先给vip办理，normal只能在没有vip等待的时候才能过去领取奖励。在这个时候，我们如果把每个人都换做线程的话，其实在这种排队的情况下，我们只需要唤醒vip用户，normal用户是不需要唤醒的，虽然我们可以拿程序进行判断，normal用户唤醒之后还会不满足条件继续等待，但是还会造成资源浪费。

- 欺骗性唤醒问题，等待线程也可能在没有其他任何线程的情况执行Object.notify()/Object.notify()下被唤醒，由于欺骗性唤醒的作用，等待线程没有任何线程对共享变量进行更新。可见，欺骗性唤醒也会导致过早唤醒。所以，在使用Object.wait()的时候，一定要放在一个魂环语句之中，欺骗性唤醒就不会对我们造成实际的影响。


### wait/notify的问题 ###

根据刚才总结，发现wait()有两个比较明显的缺点，第一个就是，执行wait(long)的线程，我们不知道是超时之后自动被取消等待的，还是别的线程执行notify()方法来唤醒的。第二点，就是我们在过早唤醒的时候举个例子，我们唤醒了一些判断条件不符合的线程，最后它们还是会因为条件不满足而继续等待。那么这两个问题能否解决呢，使用java.util.concurrent.locks.Condition类可以解决这两个问题，不过他是配合ReentrantLock可冲入锁来使用的。
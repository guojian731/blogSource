---
title: java多线程-join()
date: 2017-12-11 12:30:14
tags: java多线程
---
# join()使用介绍
**join()方法是使线程强制执行，在线程A中，创建了线程b，b.start(）的时候，那么a，b两个线程都会运行。但是如果b线程使用了b.join()，那么就会强制执行b线程，并且将A线程挂起。**

# 场景1：

**三个线程t1,t2,t3三个线程，确保运行t1结束后运行t2，在t2运行结束后运行t3。**

```java
public class ThreadTest {
    public static void main(String[] args) {
        Thread t1=new Thread(new Runnable() {
            @Override
            public void run() {
                System.out.println(Thread.currentThread().getName());
            }
        },"t1");
        try {
            t1.join();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        Thread t2=new Thread(new Runnable() {
            @Override
            public void run() {
                try {
                    t1.join();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                System.out.println(Thread.currentThread().getName());
            }
        },"t2");

        Thread t3=new Thread(new Runnable() {
            @Override
            public void run() {
                try {
                    t2.join();
                    System.out.println(Thread.currentThread().getName());
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                System.out.println(Thread.currentThread().getName());
            }
        },"t3");
        t3.start();
        t2.start();
        t1.start();
        
    }
}
```

**输出结果：t1,t2,t3**

三个线程使用启动后，如果t3先一步执行，那么在t3的run()方法里使用了t2.join(),强迫t2执行并使t3阻塞，当t2运行结束后，t3才会进行修改。

## 延伸：
**假如我们想在这个基础上三个线程都运行后在运行主线程。**

即在t1.start()方法后在加一个
``` java
System.out.println("主线程运行结束");
```
单纯的加，结果为：
主线程结束
t1
t2
t3

说明了主线程是先运行结束的（当然结果也可能是其他的，主线程也可能是后结束的），我们如何保证他的顺序呢。

我们可以在 t1.start();和"主线程运行结束"之间加上t1.join()呢？我们想到，那么当运行t1.join()的时候，确保了先执行t1()，之后再执行主线程，所以保证了t1和主线程的顺序，但是t2，t3我们是没有办法确保他们与主线程之间的先后顺序。所以在加上t2,t3的join方法就会确保三个线程都执行完毕后在执行主线程。不过有更好的方法，因为我们再以上的方法中已经确定了t1,t2,t3的执行顺序，那么只要是确保t3线程能够阻塞主线程，那么就可以完成t1-t2-t3-"主线程运行结束"这个顺序了。代码如下：
``` java
public class ThreadTest {
    public static void main(String[] args) {
        Thread t1=new Thread(new Runnable() {
            @Override
            public void run() {
                System.out.println(Thread.currentThread().getName());
            }
        },"t1");
        try {
            t1.join();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        Thread t2=new Thread(new Runnable() {
            @Override
            public void run() {
                try {
                    t1.join();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                System.out.println(Thread.currentThread().getName());
            }
        },"t2");

        Thread t3=new Thread(new Runnable() {
            @Override
            public void run() {
                try {
                    t2.join();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                System.out.println(Thread.currentThread().getName());
            }
        },"t3");
        t3.start();
        t2.start();
        t1.start();
        try {
            t3.join();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println("主线程结束");
    }
}
```

# 场景2：
**假如我们有一个很大的arrayList，里面存放的User类型，要对list进行操作，那么如果不用多线程，执行时间很长，我们可以用join()方法来解决。假设我们要讲User的value都执行*10操作。**

创建User类
``` java
public class User {

    private String name;

    private Integer value;

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public Integer getValue() {
        return value;
    }

    public void setValue(Integer value) {
        this.value = value;
    }
}
```

创建线程类
``` java 
public class ThreadT implements Runnable{
    private List<User> list;

     ThreadT(List<User> list){
        this.list=list;
    }
    @Override
    public void run() {

        for (User user:list) {
            System.out.println("执行线程为"+Thread.currentThread().getName());
            user.setValue(user.getValue()*10);
        }

    }

}
```
测试类：
``` java
public static void main(String[] args) {
        System.out.println("主线程开始");
        List<User> list=new ArrayList<User>();
        for (int i=0;i<500;i++){
            User user=new User();
            user.setName("user"+i);
            user.setValue(i);
            list.add(user);
        }
        Thread [] threads=new Thread[5];
        for (int i = 0; i < threads.length; i++) {
            threads[i]=new Thread(new ThreadT(list.subList(i*100,(i+1)*100)),"t"+i);
            threads[i].start();
        }
        for (int i = 0; i < threads.length; i++) {
            try {
                threads[i].join();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
        for (User user:list) {
            System.out.println(user.getName()+"-----"+user.getValue());
        }
        System.out.println("主线程结束");

    }
```
根据打印的输出语句：我们是并发执行了四个线程，四个线程都join(),所以当四个线程都运行结束后，主线程才会循环list，数据均是正确的。

## 疑问：
**为什么threads[i].join()不写在threads[i].start()后呢，为什么还要重新写一个for循环呢，这样不是很麻烦么？**

原因：如果threads[i].join()放在threads[i].start()后，当程序运行到threads[i].start()时，只启动了t1，然后t1.join(),这样主线程就被阻塞了，所以等到t1线程运行之后，才运行t2，然后又阻塞了主线程....没有达到t1~t4同步执行。

---
title: java多线程-线程池
date: 2018-06-13 18:01:46
tags: java多线程
---

线程是一种昂贵的资源，其开销主要包括以下几个方面：

线程的创建与启动的开销。与普通的对象相比，Java线程还占用了额外的存储空间--栈空间。并且，线程的启动会产生相应的线程调度开销。
线程的销毁,线程的销毁也有其开销。
线程调度的开销。线程的调度会导致上下文切换，从而增加处理器资源的消耗，使得应用程序本身可以使用的处理器资源减少。
一个系统能够穿件的线程总是受限于该系统所拥有的处理器数目，无论是CPU密集型还是I/O密集型线程，这些线程的数量的临界值总是处理器的数目。

因此，从整个系统乃至整个主机的角度来看，我们需要一种有效线程的方式。线程池就是有效使用线程的一种方式。

常见的对象池（比如数据库连接池）的实现方式是对象池（本身也是个对象）内部威虎一定数量的对象，客户端需要一个对象的时候就向对象池申请一个对象，用完之后再将该对象返还给对象池，于是对象池中的一个对象就可以先后为多个客户端线程服务。线程池本身也是一个对象，不过他的实现方式与普通的对象池不同，线程池内部可以预先创建一定数量的工作者线程，客户端代码并不需要向线程池借用线程而是将其需要执行的任务作为一个对象提交给线程池，线程池可能将这些任务缓存在队列（工作队列）中，而线程池内部的各个工作者线程则不断地从队列中去除任务并执行。因为，线程池可以看做给予生产者-消费者模式的一种服务，该服务内部维护的工作者线程相当于消费者线程，线程池的客户端线程相当于生产者线程，客户端代码提交给线程池的任务相当于"产品",线程池内部用于缓存任务的队列相当于传输通道。方
java.util.concurrent.ThreadPoolExecutor类就是一个线程池，客户端可以调用submit()或者execute()方法向其提交任务。



public ThreadPoolExecutor(int corePoolSize, int maximumPoolSize, long keepAliveTime, TimeUnit unit, BlockingQueue<Runnable> workQueue, ThreadFactory threadFactory, RejectedExecutionHandler handler) ;

线程池的有关线程的数量的变量总共有三个，当前的线程数量（currentPoolSize）表示线程中实际存在的工作者线程的数量。最大线程池的大小（maximumPoolSize）表示线程池中允许存在的最大线程池的数量。核心线程数（corePoolSize）表示一个不大于最大线程池大小的工作者线程数量上限。

workQueue:workQueue是被称为工作队列的阻塞队列，它相当于声场这-消费者模式中的传输通道。BlockingQueue是一个接口，在建立线程池中，可以用LinkedBlockingQueue和SynchronousQueue两个实现类，使用不同的实现类有着不同的效果。

threadFactory指定用于创建工作者线程的线程工厂（比如你向让你创建的线程都有特色，比如是守护线程，优先级等，如果你的线程没有这么多限制的话，可以缺省它）。

keepAliveTime和unit共同组成了线程池中空闲线程的最大存活时间。

handler是我们现在需要介绍的。

  实现类           所实现的处理策略   

ThreadPoolExecutor.AbortPolicy  直接抛出异常
ThreadPoolExecutor.DiscardPolicy 丢弃当前被拒绝的任务（不抛出异常）
ThreadPoolExecutor.DiscardOldestPolicy 当工作队列中最老的任务丢弃，然后重新尝试接纳被拒绝的任务  
ThreadPoolExecutor.CallerRunsPolicy 在客户端线程中执行被拒绝的任务 

我们先说一下当前线程数(currentPoolSize)，核心线程数（corePoolSize）,最大线程数(maximumPoolSize)以及工作队列（workQueue）之间的关系和联系，我们用例子可能更容易理解一些，刚开始当前线程数(currentPoolSize)为0，核心线程数（corePoolSize）为10，最大线程数设置为100，工作队列为50。

1.当一下有15个任务的时候，当前线程数将会升到10个。因为没有超过最大队列的容量，所以就不会再增加线程，之后发现5s后线程有空闲的情况，就会降空闲的线程除掉，之后线程池的数量会变为10.
2.当一下有70个任务的时候，当前线程数将会升到10个，因为工作队列被装满和当前10个工作者线程都被装满，还有10个线程无法处理，会再新建10个线程，当前线程数为20个，之后发现5s后线程有空闲的情况，就会降空闲的线程除掉，之后线程池的数量会变为10.
3.当一下有100个任务的时候，当前线程数将会升到10个，因为工作队列被装满和当前10个工作者线程都被装满，还有40个线程无法处理，会再新建40个线程，当前线程数为50个，之后发现5s后线程有空闲的情况，就会降空闲的线程除掉，之后线程池的数量会变为10.
4.当一下有200个任务的时候，当前线程数将会升到10个，因为工作队列被装满和当前10个工作者线程都被装满，还有140个线程无法处理，因为最大线程池中只能有100个线程，还有50个线程在外面无法处理。这时候就需要用到handler了。
 1）ThreadPoolExecutor.AbortPolicy：直接抛出异常 
 2）ThreadPoolExecutor.DiscardPolicy：丢弃当前被拒绝的任务（不抛出异常），也就是最后的50条直接被拒绝不处理，最终只处理150条。
 3）ThreadPoolExecutor.DiscardOldestPolicy：将队列中最老的没处理的任务抛弃，在队列中接收新的，最终只处理150条。
 4）ThreadPoolExecutor.CallerRunsPolicy：在客户端线程中执行被拒绝的任务，，最终只处理200条都执行。

这是我们将队列为LinkedBlockingQueue的时候，如果我们的队列为SynchronousQueue的话，其实我们的队列实际上是没有容量的，相当于的容量为0；



这几个参数关联性非常高，注意，是"非常"，为什么这么强调呢，看我下面的解释。
我们默认我们的线程池是新的，里面线程一个都没有，即当前线程数currentPoolSize=0.接下来我们在进行一些模拟，我将以corePoolSize代表核心线程数，workQueue代表等待的队列。keepAliveTime和unit组成线程空闲了5s就销毁。
1.corePoolSize=5，maximumPoolSize=8，workQueue为10，任务数为9
9个任务队列刚好能放在队列里，而且currentPoolSize<corePoolSize,这个时候currentPoolSize就会涨到5，当线程完成任务后，连续5s都空闲的时候，会被检查，发现池内的线程数刚好等于corePoolSize，不会销毁。
2.corePoolSize=5，maximumPoolSize=8，workQueue为10，任务数为2
2个任务可以存储到队列里，而且currentPoolSize<corePoolSize,这个时候currentPoolSize就会涨到2，当线程完成任务之后，连续5s都空闲，被检测，发现线程数不超过5个，进行保留，不销毁。
3.corePoolSize=5，maximumPoolSize=8，workQueue为10，任务数为20
20个任务而队列只能放下容量为10的队列，所以currentPoolSize会涨到8（maximumPoolSize最大线程数）来处理任务，当线程完成任务后，连续5s后空闲后，发现maximumPoolSize>corePoolSize,所以线程数降到5个，剩下的3个进行销毁。
4.corePoolSize=0，maximumPoolSize=8，workQueue为100，任务数为20
20个任务可以存储到队列里，这时候corePoolSize=0，所以currentPoolSize=1，无法建立新的线程，那么这20个任务只能串行执行。当任务都执行过后5s后，当前线程池的数量是1，要销毁变成0。
5.corePoolSize=0，maximumPoolSize=8，workQueue为10，任务数为30
30个任务数无法存储到队列，这时候currentPoolSize会涨到8，执行完过后5s后，当前线程数为8，如果都空闲，就会都销毁。

我们现在就来做测试。简单做一个线程，使用随机数控制在1900~2100内，模拟业务逻辑需要在2s左右，如果随机数可以被2整除，返回"true",不能返回"false"。

``` java 
public class RandomThread implements Callable<String> {
    @Override
    public String call() throws Exception {
        Random rand = new Random();
        int i=rand.nextInt(200)+1900;
        try {
            Thread.sleep(i);
            System.out.println("这个线程执行了,i的值为："+i);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        if (i%2==0){
            return "true";
        }else{
            return "false";
        }
    }
}

```

``` java 
public class Test {
    static int length=20;
    static int corePoolSize=4;
    static int  workQueue=10;
    final static int N_CPU =Runtime.getRuntime().availableProcessors();
    final static ThreadPoolExecutor executor=new ThreadPoolExecutor(corePoolSize,N_CPU*2,5, TimeUnit.SECONDS,new ArrayBlockingQueue<Runnable>(workQueue),new ThreadPoolExecutor.CallerRunsPolicy());
    public static void main(String[] args) {
        
        int count=0;
        long start=System.currentTimeMillis();
        List<Future> list =new ArrayList<>(length);
        for (int i=0;i<length;i++){
            Future<String> future =executor.submit(new RandomThread());
            list.add(future);
        }
        
        for (Future future:list){
            try {
                future.get();
            } catch (InterruptedException e) {
                e.printStackTrace();
            } catch (ExecutionException e) {
                e.printStackTrace();
            }
        }

        System.out.println("完成所有"+length+"个任务之后线程数量为："+executor.getPoolSize());
        System.out.println("所有任务完成耗时："+(System.currentTimeMillis()-start));    
        try {
            Thread.sleep(6000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println("完成任务6s后线程池的线程数量为："+executor.getPoolSize());
    }
}

```

场景1：
这个线程执行了,i的值为：1943
这个线程执行了,i的值为：1948
这个线程执行了,i的值为：1954
这个线程执行了,i的值为：2061
这个线程执行了,i的值为：2090
这个线程执行了,i的值为：1917
这个线程执行了,i的值为：1994
这个线程执行了,i的值为：2043
这个线程执行了,i的值为：2030
完成所有9个任务之后线程数量为：5
所有任务完成耗时：4094
完成任务6s后线程池的线程数量为：5

场景2：
这个线程执行了,i的值为：2038
这个线程执行了,i的值为：2043
完成所有2个任务之后线程数量为：2
所有任务完成耗时：2046
完成任务6s后线程池的线程数量为：2

场景3：
这个线程执行了,i的值为：1903
这个线程执行了,i的值为：1916
这个线程执行了,i的值为：1925
这个线程执行了,i的值为：1932
这个线程执行了,i的值为：1933
这个线程执行了,i的值为：1957
这个线程执行了,i的值为：1962
这个线程执行了,i的值为：2015
这个线程执行了,i的值为：2037
这个线程执行了,i的值为：1912
这个线程执行了,i的值为：1967
这个线程执行了,i的值为：2046
这个线程执行了,i的值为：2016
这个线程执行了,i的值为：2061
这个线程执行了,i的值为：2038
这个线程执行了,i的值为：2092
这个线程执行了,i的值为：2082
这个线程执行了,i的值为：1981
这个线程执行了,i的值为：1941
这个线程执行了,i的值为：2064
完成所有20个任务之后线程数量为：8
所有任务完成耗时：5937
完成任务6s后线程池的线程数量为：5


场景4：

这个线程执行了,i的值为：1956
这个线程执行了,i的值为：2072
这个线程执行了,i的值为：2008
这个线程执行了,i的值为：1920
这个线程执行了,i的值为：2024
这个线程执行了,i的值为：1920
这个线程执行了,i的值为：1928
这个线程执行了,i的值为：2012
这个线程执行了,i的值为：2049
这个线程执行了,i的值为：2020
这个线程执行了,i的值为：2046
这个线程执行了,i的值为：1980
这个线程执行了,i的值为：2094
这个线程执行了,i的值为：1905
这个线程执行了,i的值为：1991
这个线程执行了,i的值为：1918
这个线程执行了,i的值为：2021
这个线程执行了,i的值为：2012
这个线程执行了,i的值为：2087
这个线程执行了,i的值为：2059
完成所有20个任务之后线程数量为：1
所有任务完成耗时：40039
完成任务6s后线程池的线程数量为：0

场景5 

这个线程执行了,i的值为：1900
这个线程执行了,i的值为：1956
这个线程执行了,i的值为：1979
这个线程执行了,i的值为：1994
这个线程执行了,i的值为：2034
这个线程执行了,i的值为：2040
这个线程执行了,i的值为：2073
这个线程执行了,i的值为：2088
这个线程执行了,i的值为：2091
这个线程执行了,i的值为：2027
这个线程执行了,i的值为：1955
这个线程执行了,i的值为：2061
这个线程执行了,i的值为：1988
这个线程执行了,i的值为：1965
这个线程执行了,i的值为：2064
这个线程执行了,i的值为：2008
这个线程执行了,i的值为：2058
这个线程执行了,i的值为：2086
这个线程执行了,i的值为：1979
这个线程执行了,i的值为：2058
这个线程执行了,i的值为：1966
这个线程执行了,i的值为：1944
这个线程执行了,i的值为：1989
这个线程执行了,i的值为：2061
这个线程执行了,i的值为：2094
这个线程执行了,i的值为：2086
这个线程执行了,i的值为：2040
这个线程执行了,i的值为：1999
这个线程执行了,i的值为：2029
这个线程执行了,i的值为：2075
完成所有30个任务之后线程数量为：8
所有任务完成耗时：8074
完成任务6s后线程池的线程数量为：0


我们之前一直没有说我们的handler，ThreadPoolExecutor提供的RejectedExecutionHander实现类

ThreadPoolExecutor.AbortPolicy            直接抛出异常
ThreadPoolExecutor.DiscardPolicy          丢弃当前被拒绝的任务（而不抛出任何异常）
ThreadPoolExecutor.DiscardOldestPolicy    将工作队列中最老的任务丢弃，然后重新尝试被拒绝的任务
ThreadPoolExecutor.CallerRunsPolicy       在客户端线程中执行被拒绝的任务

这个就是实际上任务的队列超过了线程池存储的规定值时，会发生的策略。

















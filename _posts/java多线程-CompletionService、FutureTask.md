---
title: java多线程-CompletionService、FutureTask
date: 2018-06-28 17:57:48
tags: java多线程
---

## CompletionService使用 ##

### 案例 ###
之前我们使用了线程池，发现线程池在多线程的使用上非常的好用，但是有一点弊端，是什么呢？我们来用代码看一下。我们现在有一个核心线程数是5个的线程池，现在一下来了6个任务，我们用Thread.sleep()来模拟中间运行时长，我们假设i%5==0的线程刚好发生网络延迟，需要10000ms，其他的线程很快，只需要1000ms。我们来看一下结果。

``` java 
public class Test2 {
    static ExecutorService executor= new ThreadPoolExecutor(5, 5, 0L, TimeUnit.MILLISECONDS, new LinkedBlockingQueue(20),new ThreadPoolExecutor.CallerRunsPolicy());
    public static void main(String[] args) {
        List<Future<String[]>> futureList=new ArrayList<>();
        long start=System.currentTimeMillis();
        for(int i=0;i<6;i++){
            Future<String[]> future= executor.submit(new ThreadTest1(i));
            futureList.add(future);
        }
        for (Future<String[]> future:futureList){
            try {
                System.out.println(future.get()[2]+"序号"+future.get()[0]+"实际处理时间:"+future.get()[1]+";实际所用时间："+(System.currentTimeMillis()-start)+"ms");
                //获取到future之后的逻辑处理
                //Thread.sleep(1000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            } catch (ExecutionException e) {
                e.printStackTrace();
            }
        }
        System.out.println("总耗时:"+(System.currentTimeMillis()-start)+"ms");
    }
    
    static class  ThreadTest1 implements Callable<String[]> {
        private int num;
        public ThreadTest1(int num){
            this.num=num;
        }
        @Override
        public String[] call() throws Exception {
            if (num % 5==0){
                Thread.sleep(10000);
                return new String[]{String.valueOf(num),String.valueOf(5000),Thread.currentThread().getName()};
            }else {
                Thread.sleep(1000);
                return new String[]{String.valueOf(num),String.valueOf(1000),Thread.currentThread().getName()};
            }
        }
    }
}

```

``` java 
pool-1-thread-1序号0实际处理时间:10000;实际所用时间：10002ms
pool-1-thread-2序号1实际处理时间:1000;实际所用时间：10002ms
pool-1-thread-3序号2实际处理时间:1000;实际所用时间：10002ms
pool-1-thread-4序号3实际处理时间:1000;实际所用时间：10002ms
pool-1-thread-5序号4实际处理时间:1000;实际所用时间：10002ms
pool-1-thread-2序号5实际处理时间:10000;实际所用时间：11001ms
总耗时:11001ms
```

**我们可以看到，其实线程2、3、4、5实际处理的时间都是1000ms，但是因为线程1的future最先进入futureList中，所以线程1的future.get()阻塞了其他的线程。当线程1的future.get()处理完之后，其他线程等待的线程立马可以获取结果了。而且我们看i=5时，会发现，其实线程4在处理完任务1000ms后返回future的值之后就立马返回线程池中，供其他请求使用，所以序号5的实际所用时间是11001ms而不是20000ms。这样看我们觉得没什么太大问题，因为这种情况只是结果阻塞，而线程并没有一起等待，还在实际工作中。但是我们现实生活中，经常会拿到结果future结果后再进行后续操作，假设操作时间为1000ms，我们把上面的Thread.sleep(1000);注释打开，再运行一下，我们使用了16s完成了所有的操作，仔细想一想，如果序号1、2、3、4执行完之后就进行业务处理，可以省下4s的时间，怎么样才能解决这个问题呢？**

``` java 
pool-1-thread-1序号0实际处理时间:10000;实际所用时间：10002ms
pool-1-thread-2序号1实际处理时间:1000;实际所用时间：11002ms
pool-1-thread-3序号2实际处理时间:1000;实际所用时间：12002ms
pool-1-thread-4序号3实际处理时间:1000;实际所用时间：13003ms
pool-1-thread-5序号4实际处理时间:1000;实际所用时间：14003ms
pool-1-thread-4序号5实际处理时间:10000;实际所用时间：15003ms
总耗时:16004ms
```

### CompletionService ###

CompletionService是一个接口，他的核心方法是:
``` java 
	Future<V> take() throws InterruptedException;

    Future<V> poll();

    Future<V> poll(long var1, TimeUnit var3) throws InterruptedException;
```

其中ExecutorCompletionService实现take()方法的时候，维护了一个队列，将已经处理完的结果放进队列里，取出来的时候按照该队列的顺序来进行，而不是之前的放进去的顺序。如果队列中没有数据，会进行阻塞。建立ExecutorCompletionService的时候我们要传入一个线程池。好了，我们将代码稍稍改变一下。

``` java 

public class Test11 {

    static ExecutorService executor= Executors.newFixedThreadPool(5);
    static CompletionService<String[]> cService = new ExecutorCompletionService<String[]>(executor);
    
    
    
    public static void main(String[] args) {
        
        for(int i=0;i<6;i++){
            cService.submit(new ThreadTest1(i));
        }
        long start=System.currentTimeMillis();
        for (int i=0;i<6;i++){
            try {
               Future<String[]> future= cService.take();
                System.out.println(future.get()[2]+"线程"+future.get()[0]+"实际处理时间:"+future.get()[1]+";实际所用时间："+(System.currentTimeMillis()-start)+"ms");
                //获取到future之后的逻辑处理
                //Thread.sleep(1000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            } catch (ExecutionException e) {
                e.printStackTrace();
            }
        }
        System.out.println("总耗时:"+(System.currentTimeMillis()-start)+"ms");
    }
    
    static class  ThreadTest1 implements Callable<String[]> {
        private int num;
        public ThreadTest1(int num){
            this.num=num;
        }
        @Override
        public String[] call() throws Exception {
            if (num % 5==0){
                Thread.sleep(10000);
                return new String[]{String.valueOf(num),String.valueOf(10000),Thread.currentThread().getName()};
            }else {
                Thread.sleep(1000);
                return new String[]{String.valueOf(num),String.valueOf(1000),Thread.currentThread().getName()};
            }
        }
    }
}

```

我们看到结果，先完成的结果在主线程进行处理了，不会因为没完成的future阻塞完成的future执行了，这样主线程的利用率就变的高效了。
``` java 

pool-1-thread-4线程3实际处理时间:1000;实际所用时间：1000ms
pool-1-thread-3线程2实际处理时间:1000;实际所用时间：2001ms
pool-1-thread-2线程1实际处理时间:1000;实际所用时间：3002ms
pool-1-thread-5线程4实际处理时间:1000;实际所用时间：4003ms
pool-1-thread-1线程0实际处理时间:10000;实际所用时间：10000ms
pool-1-thread-2线程5实际处理时间:10000;实际所用时间：11001ms
总耗时:12001ms

```


## FutureTask使用 ##

ExecutorService这个接口，我们需要不需要返回结果的时候,需要的参数必须继承Runnable接口。当我们需要返回结果的时候，必须需要继承Callable接口，而且又能继承future接口代表返回值。我们不能因为返回不返回的问题就继承两个接口，并且里面的核心代码一样吧？有什么解决办法吗？有，它就是FutureTask，他继承了Runnable接口和future接口，而且你可以传入参数Callable实现，他会转成Runnable接口的实现。

``` java 
public class DemoFuture implements Callable {
    private int i;
    public DemoFuture(int i){
        this.i=i;
    }
    
    @Override
    public String call() throws Exception {
        System.out.println(i);
        return "执行的是:"+i;
    }
}

```


``` java 
public class Test1 {
    static ExecutorService executor=Executors.newFixedThreadPool(10);
    public static void main(String[] args) {
        executor.execute(new FutureTask<>(new DemoFuture(2)));
    }
}
```




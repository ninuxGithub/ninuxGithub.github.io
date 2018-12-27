---
title: ThreadPoolExecutor的使用 
author: ninuxGithub
layout: post
date: 2018-12-27 17:30:51
description: "ThreadPoolExecutor的使用"
tag: java
---


## ThreadPoolExecutor类的介绍
    ThreadPoolExecutor 是java集合包下java.util.concurrent， 对外提供4个构造函数

```java
    public ThreadPoolExecutor(int corePoolSize,int maximumPoolSize,long keepAliveTime,TimeUnit unit,BlockingQueue<Runnable> workQueue) 
    public ThreadPoolExecutor(int corePoolSize,int maximumPoolSize,long keepAliveTime,TimeUnit unit,BlockingQueue<Runnable> workQueue,ThreadFactory threadFactory) 
    public ThreadPoolExecutor(int corePoolSize,int maximumPoolSize,long keepAliveTime,TimeUnit unit,BlockingQueue<Runnable> workQueue,RejectedExecutionHandler handler) 
    public ThreadPoolExecutor(int corePoolSize,int maximumPoolSize,long keepAliveTime,TimeUnit unit,BlockingQueue<Runnable> workQueue,ThreadFactory threadFactory,RejectedExecutionHandler handler) 
```
    coreSize: 核心池的大小，默认情况不创建线程，等待任务到来的时候在创建线程。
    但是如果调用了prestartAllCoreThreads() or prestartCoreThread() 就会触发在任务到来之前
    创建线程，等待任务的到来。
    默认情况的coreSize为0 ， 当线程池中的线程的数量达到coreSize就会将任务丢到任务队列workQueue.
    
    maximumPoolSize: 最大的线程的数量。
    
    keepalive: 当线程的数量大于coreSize的时候 触发了创建线程到达maxSize, 线程多了后执行任务
    随之是每个线程的任务会减少，导致有些线程是处于空闲状态的，那么就keepalive就会起作用， 让线程
    进入一个等待的状态， 等待任务多的时候在开始工作。
    
    handler:
        rejectPolicy: 有四种拒绝策略
        1.CallerRunsPolicy:不放弃任务，调用线程的run来执行任务，减缓新任务的提交速度
        2.AbortPolicy: 直接丢弃任务，并且抛出异常RejectedException
        3.DsicardPolicy: 丢弃任务，但是不抛异常
        4.DiscardOldestPolicy: 丢弃队列里面最前面的任务，处理新的任务
    
    
    unit: keepalive的时间单位  请查看TimeUtit
    
    workQueue: 阻塞的队列， ArrayBlockingQueue, LinkedBlockingQueue, SynchronizedQueue, PriorityBlockingQueue;
    
    threadFactory: 线程工程， 创建线程的类
    
    
    执行任务的使用调用的方法：
    execute: 提交任务到线程池去执行。
    submit: 也是提交任务到线程池， 但是submit 可以返回运算的结果，采用了Futrue来获取结果。

## execute是如何运行的呢？
      ThreadPoolExecutor.Worker 的定义是：private final class Worker extends AbstractQueuedSynchronizer implements Runnable
      是一个Runnable 重新了run 方法 
      1.创建Worker  ThreadPoolExecutor.execute ---> ThreadPoolExecutor.addWorder(包含一个线程)
      2.线程开启： run---->ThreadPoolExecutor.this.runWorker(this)
      3.获取一个任务：--->var3 = this.getTask() 从队列获取一个任务 
      4.执行任务：  -->var3.run();
      
      任务执行完毕~~~
      
## submit 是如何运行的呢？
   代码如下：  submit(Runnable) 是调用的execute 方法
   
```java
public Future<?> submit(Runnable task) {
    if (task == null) throw new NullPointerException();
    RunnableFuture<Void> ftask = newTaskFor(task, null);
    execute(ftask);
    return ftask;
}
``` 

   submit(Callable) 的运行原理如下：
   AbstractExecutorService.submit()---> 创建一个FutureTask  -->FutureTask.call  
   当调用future.get()的时候会导致阻塞，怎么阻塞的呢？
   源代码如下：
   
   run 运行的时候计算的时间比较长 ， 当我们在调用future.get()  ---> awaitDone-->park住
   当run运行完毕后  --->调用 set方法会调用--->finishCompletion  unpark 解除了阻塞然
   然后get就可以获取结果了

```java

public void run() {
    if (state != NEW ||
        !UNSAFE.compareAndSwapObject(this, runnerOffset,
                                     null, Thread.currentThread()))
        return;
    try {
        Callable<V> c = callable;
        if (c != null && state == NEW) {
            V result;
            boolean ran;
            try {
                result = c.call();
                ran = true;
            } catch (Throwable ex) {
                result = null;
                ran = false;
                setException(ex);
            }
            if (ran)
                set(result);
        }
    } finally {
        // runner must be non-null until state is settled to
        // prevent concurrent calls to run()
        runner = null;
        // state must be re-read after nulling runner to prevent
        // leaked interrupts
        int s = state;
        if (s >= INTERRUPTING)
            handlePossibleCancellationInterrupt(s);
    }
}
public V get() throws InterruptedException, ExecutionException {
    int s = state;
    if (s <= COMPLETING)
        s = awaitDone(false, 0L);
    return report(s);
}


private int awaitDone(boolean timed, long nanos) throws InterruptedException {
    final long deadline = timed ? System.nanoTime() + nanos : 0L;
    WaitNode q = null;
    boolean queued = false;
    for (;;) {
        if (Thread.interrupted()) {
            removeWaiter(q);
            throw new InterruptedException();
        }

        int s = state;
        if (s > COMPLETING) {
            if (q != null)
                q.thread = null;
            return s;
        }
        else if (s == COMPLETING) // cannot time out yet
            Thread.yield();
        else if (q == null)
            q = new WaitNode(); //初始化等待的节点
        else if (!queued)
            queued = UNSAFE.compareAndSwapObject(this, waitersOffset,
                                                 q.next = waiters, q);
        else if (timed) {
            nanos = deadline - System.nanoTime();
            if (nanos <= 0L) {
                removeWaiter(q);
                return state;
            }
            LockSupport.parkNanos(this, nanos);
        }
        else
            LockSupport.park(this); // park 住了 阻塞在这里 ， 那么什么时候唤醒呢？
            
            
            //一个unpark,可以抵消过去的一个park或者将来的一个park。
            //多个unpark,可以抵消过去的多个park。
            //多个unpark,不可以抵消将来的多个park。
        
    }
}



//在run方法执行完毕后  会调用set方法--->在调用finishCompletion
private void finishCompletion() {
    // assert state > COMPLETING;
    for (WaitNode q; (q = waiters) != null;) {
        if (UNSAFE.compareAndSwapObject(this, waitersOffset, q, null)) {
            for (;;) {
                Thread t = q.thread;
                if (t != null) {
                    q.thread = null;
                    LockSupport.unpark(t);   //在run 执行完毕后  在这个地方做了unpark 解除了阻塞
                }
                WaitNode next = q.next;
                if (next == null)
                    break;
                q.next = null; // unlink to help gc
                q = next;
            }
            break;
        }
    }

    done();

    callable = null;        // to reduce footprint
}

```   




## 测试类如下

```java
public class ThreadPoolTest {

    //static ExecutorService executor = Executors.newFixedThreadPool(5);
    static ThreadPoolExecutor executor = new ThreadPoolExecutor(5, 8, 200,
            TimeUnit.MILLISECONDS, new ArrayBlockingQueue<Runnable>(5), new ThreadPoolExecutor.CallerRunsPolicy());

    public static void main(String[] args) {
        Task task = new Task("java");
        for (int i = 0; i < 100; i++) {
            executor.execute(task);
            System.out.println("poolSize:" + executor.getPoolSize() + " 等待执行任务的数量：" + executor.getQueue().size() + " 执行完毕的任务量：" + executor.getCompletedTaskCount());
        }

        executor.shutdown();
    }

}


class Task implements Runnable {

    private String name;

    public Task(String name) {
        this.name = name;
    }

    @Override
    public void run() {
        try {
            Thread.sleep(3000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println(Thread.currentThread().getName() + " " + name);
    }
}



public class App {
	static ThreadPoolExecutor executor = new ThreadPoolExecutor(5, 8, 200, TimeUnit.MILLISECONDS,
			new ArrayBlockingQueue<Runnable>(5), new ThreadPoolExecutor.CallerRunsPolicy());

	public static void main(String[] args) {

		System.out.println("开始计算");
		Future<String> future = executor.submit(new Task());
		try {
			boolean isDone = future.isDone();
			while (!isDone) {
				isDone = future.isDone();
				System.out.println("在计算中.....");
				Thread.sleep(100);
			}
			System.out.println(future.get());
			System.out.println("计算完毕");
		} catch (InterruptedException e) {
			e.printStackTrace();
		} catch (ExecutionException e) {
			e.printStackTrace();
		}
		executor.shutdown();
	}
}

class Task implements Callable<String> {

	public String call() throws Exception {
		TimeUnit.MINUTES.sleep(5);
		return "java";
	}

}








```   

         
      

        
    
---
title: TransmittableThreadLocal使用
author: ninuxGithub
layout: post
date: 2019-1-9 11:10:45
description: "TransmittableThreadLocal使用"
tag: java
---

### 问题分析
ThreadLocal: 解决多线程变量之间的隔离， 每个线程对变量做了一个副本

InheriableThreadLocal: 解决了父线程变量复制到子线程的问题

```java
class Thread{
    //...略
    private void init(ThreadGroup g, Runnable target, String name,
                          long stackSize, AccessControlContext acc) {
        if (name == null) {
            throw new NullPointerException("name cannot be null");
        }

        this.name = name.toCharArray();

        Thread parent = currentThread();
        SecurityManager security = System.getSecurityManager();
        if (g == null) {
            /* Determine if it's an applet or not */

            /* If there is a security manager, ask the security manager
               what to do. */
            if (security != null) {
                g = security.getThreadGroup();
            }

            /* If the security doesn't have a strong opinion of the matter
               use the parent thread group. */
            if (g == null) {
                g = parent.getThreadGroup();
            }
        }

        /* checkAccess regardless of whether or not threadgroup is
           explicitly passed in. */
        g.checkAccess();

        /*
         * Do we have the required permissions?
         */
        if (security != null) {
            if (isCCLOverridden(getClass())) {
                security.checkPermission(SUBCLASS_IMPLEMENTATION_PERMISSION);
            }
        }

        g.addUnstarted();

        this.group = g;
        this.daemon = parent.isDaemon();
        this.priority = parent.getPriority();
        if (security == null || isCCLOverridden(parent.getClass()))
            this.contextClassLoader = parent.getContextClassLoader();
        else
            this.contextClassLoader = parent.contextClassLoader;
        this.inheritedAccessControlContext =
                acc != null ? acc : AccessController.getContext();
        this.target = target;
        setPriority(priority);
        
        
        //###############################重点##############################################
        //重点地方： 如果当前线程的inheritableThreadLocals 不为空的时候， 
        // 将父线程的inheritableThreadLocals 复制到子线程的LocalMap，从而解决了父线程变量复制到子线程的问题
        if (parent.inheritableThreadLocals != null)
            this.inheritableThreadLocals =
                ThreadLocal.createInheritedMap(parent.inheritableThreadLocals);
        /* Stash the specified stack size in case the VM cares */
        this.stackSize = stackSize;

        /* Set thread ID */
        tid = nextThreadID();
    }
}
```

那么TransmittableThreadLocal解决了什么问题呢？
如果思考这个问题的时候，注意上面的代码只有父线程里面 创建一个 线程（Thread） 的时候才回去调用 init  才会有inheritableThreadLocals的拷贝动作
如果不是建立一个线程呢， 如果是采用线程池来运行子线程的任务（runnable）呢？ inheriableThreadLocal 还可以解决问题吗？
demo如下：

```java
public class InheritableThreadLocalTestUpdate {
    static final ThreadLocal<Integer> inheritableThreadLocal = new InheritableThreadLocal<Integer>();
    static ExecutorService pool = Executors.newFixedThreadPool(2);

//    static TransmittableThreadLocal<Integer> inheritableThreadLocal = new TransmittableThreadLocal<>();
//    static ExecutorService pool = TtlExecutors.getTtlExecutorService(Executors.newFixedThreadPool(5));


    static class MainThread extends Thread {

        private int index;

        public MainThread(int index) {
            this.index = index;
        }

        @Override
        public void run() {

            inheritableThreadLocal.set(index);
            pool.execute(new Runnable() {
                @Override
                public void run() {
                    System.out.println("child inheritableThreadLocal: " + inheritableThreadLocal.get());
                }
            });
        }
    }
    public static void main(String[] args) {
        for (int i = 0; i < 100; i++) {
            new MainThread(i).start();
        }
    }
}

```
打印结果：
child inheritableThreadLocal: 1
child inheritableThreadLocal: 1
child inheritableThreadLocal: 1
child inheritableThreadLocal: 1
child inheritableThreadLocal: 1
child inheritableThreadLocal: 0
child inheritableThreadLocal: 0
child inheritableThreadLocal: 0
child inheritableThreadLocal: 0
child inheritableThreadLocal: 0
child inheritableThreadLocal: 0
child inheritableThreadLocal: 0
child inheritableThreadLocal: 0
child inheritableThreadLocal: 0
child inheritableThreadLocal: 0
child inheritableThreadLocal: 0
child inheritableThreadLocal: 0
child inheritableThreadLocal: 0
child inheritableThreadLocal: 0
child inheritableThreadLocal: 0

结果显示不是我们要的结果： 
结论： 当使用线程池来运行我们的子线程的任务的时候，采用InheriableThreadLocal是无法解决变量传递的问题的

解决办法： 采用阿里推出的transmitableThreadLocal 很好的解决了这个问题

```java
public class TransmittableThreadLocal<T> extends InheritableThreadLocal<T> {

    @Override
    public final T get() {
        T value = super.get();
        if (null != value) {
            addValue();
        }
        return value;
    }

    @Override
    public final void set(T value) {
        super.set(value);
        if (null == value) { // may set null to remove value
            removeValue();
        } else {
            //如果值不为空的时候，添加到holder
            addValue();
        }
    }

    @Override
    public final void remove() {
        removeValue();
        super.remove();
    }

    void superRemove() {
        super.remove();
    }

    T copyValue() {
        return copy(get());
    }

    private static InheritableThreadLocal<Map<TransmittableThreadLocal<?>, ?>> holder =
            new InheritableThreadLocal<Map<TransmittableThreadLocal<?>, ?>>() {
                @Override
                protected Map<TransmittableThreadLocal<?>, ?> initialValue() {
                    return new WeakHashMap<TransmittableThreadLocal<?>, Object>();
                }

                @Override
                protected Map<TransmittableThreadLocal<?>, ?> childValue(Map<TransmittableThreadLocal<?>, ?> parentValue) {
                    return new WeakHashMap<TransmittableThreadLocal<?>, Object>(parentValue);
                }
            };

    private void addValue() {
        if (!holder.get().containsKey(this)) {
            //将值放到holder
            holder.get().put(this, null); // WeakHashMap supports null value.
        }
    }
}

```  


execute 的时候怎么获取值呢?

```java
public final class TtlRunnable implements Runnable {
    private final AtomicReference<Map<TransmittableThreadLocal<?>, Object>> copiedRef;
    private final Runnable runnable;
    private final boolean releaseTtlValueReferenceAfterRun;

    private TtlRunnable(Runnable runnable, boolean releaseTtlValueReferenceAfterRun) {
        this.copiedRef = new AtomicReference<Map<TransmittableThreadLocal<?>, Object>>(TransmittableThreadLocal.copy());
        this.runnable = runnable;
        this.releaseTtlValueReferenceAfterRun = releaseTtlValueReferenceAfterRun;
    }

    /**
     * wrap method {@link Runnable#run()}.
     */
    @Override
    public void run() {
        //获取WeekReference 弱引用
        Map<TransmittableThreadLocal<?>, Object> copied = copiedRef.get();
        if (copied == null || releaseTtlValueReferenceAfterRun && !copiedRef.compareAndSet(copied, null)) {
            throw new IllegalStateException("TTL value reference is released after run!");
        }

        Map<TransmittableThreadLocal<?>, Object> backup = TransmittableThreadLocal.backupAndSetToCopied(copied);
        try {
            //like apo around , before dothing, after dothing 
            runnable.run();
        } finally {
            TransmittableThreadLocal.restoreBackup(backup);
        }
    }
}

```


### 使用如下

添加依赖

```xml
<dependency>
    <groupId>com.alibaba</groupId>
    <artifactId>transmittable-thread-local</artifactId>
    <version>2.2.0</version>
</dependency>
```

```java
package com.example.demo.test;

import com.alibaba.ttl.TransmittableThreadLocal;
import com.alibaba.ttl.threadpool.TtlExecutors;

import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

/**
 * @date 2019-1-9
 * @description 作用 多线程提交任务的时候会threadLocal导致传递的变量失效
 */
public class TransmitTableThreadLocalTest {

    static TransmittableThreadLocal<String> threadLocal = new TransmittableThreadLocal<>();
    static ExecutorService pool = TtlExecutors.getTtlExecutorService(Executors.newFixedThreadPool(5));


    //如果采用jdk的threadLocal 难以保证在线程池执行的时候变量的传递
    //static ThreadLocal<String> threadLocal = new InheritableThreadLocal<>();
    //static ThreadLocal<String> threadLocal = new ThreadLocal<>();
    //static ExecutorService pool = Executors.newFixedThreadPool(2);

    public static void main(String[] args) {


        for (int i = 0; i < 100; i++) {
            int index = i;
            pool.execute(new Thread(new Runnable() {

                @Override
                public void run() {

                    //传递值
                    threadLocal.set("java " + index);

                    pool.execute(new Thread(
                            new Runnable() {

                                @Override
                                public void run() {
                                    //获取值
                                    System.out.println(Thread.currentThread().getName() + " 获取修改后的值:" + threadLocal.get());
                                }
                            }
                    ));
                }
            }));
        }

        try {
            Thread.sleep(1000);
            pool.shutdown();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }


    }
}

```


        
---
title: ReentrantLock
author: ninuxGithub
layout: post
date: 2019-4-25 11:16:12
description: "ReentrantLock"
tag: java
---


### ReentrantLock分析
    要了解ReentrantLock必须要首先理解一下AQS(AbstractQueuedSynchronizer)的大概的内部结构
    
    aqs 内部维护了一个Node 类型的head,tail ; 
        head: 等待获取锁的首节点
        tail: 等待获取锁的尾节点
    
    Node是一个内部类；
    构造函数 Node(Thread thread, Node mode) 创建Node的时候需要一个当前的线程和线程等待的模式
    等待模式有2种： Share,Exclusive 模式
    waitStatus： 节点的等待的状态有4种状态
        1: canceld 线程的获取锁由于超时或interrupt取消了
        -1：singal 后续的节点需要被唤醒
        -2: condition 当前的节点在condition queue, 当某个时候状态被改变的(status 变为0)时候才会去condition race 获取锁
        -3：propagate 下一个acquireShared无条件的传播
        0： 默认的值
    
    如下是Node的代码， 可以理解为在aqs内部维护了一个双向的链表， 每个Node都是一个线程+模式构成的等待着排队获取锁的节点；
    通过AQS.addWaiter方法知道如果acquire失败那么会创建一个等待的节点，添加到tail后面然后设置新创建的node为tail;
    
    head(Thread,Mode)---next(Thread,Mode)----.......----tail(Thread,Mode)
    
    
```java
static final class Node {
        /** Marker to indicate a node is waiting in shared mode */
        static final Node SHARED = new Node();
        /** Marker to indicate a node is waiting in exclusive mode */
        static final Node EXCLUSIVE = null;

        /** waitStatus value to indicate thread has cancelled */
        static final int CANCELLED =  1;
        /** waitStatus value to indicate successor's thread needs unparking */
        static final int SIGNAL    = -1;
        /** waitStatus value to indicate thread is waiting on condition */
        static final int CONDITION = -2;
        /**
         * waitStatus value to indicate the next acquireShared should
         * unconditionally propagate
         */
        static final int PROPAGATE = -3;

        /**
         * Status field, taking on only the values:
         *   SIGNAL:     The successor of this node is (or will soon be)
         *               blocked (via park), so the current node must
         *               unpark its successor when it releases or
         *               cancels. To avoid races, acquire methods must
         *               first indicate they need a signal,
         *               then retry the atomic acquire, and then,
         *               on failure, block.
         *   CANCELLED:  This node is cancelled due to timeout or interrupt.
         *               Nodes never leave this state. In particular,
         *               a thread with cancelled node never again blocks.
         *   CONDITION:  This node is currently on a condition queue.
         *               It will not be used as a sync queue node
         *               until transferred, at which time the status
         *               will be set to 0. (Use of this value here has
         *               nothing to do with the other uses of the
         *               field, but simplifies mechanics.)
         *   PROPAGATE:  A releaseShared should be propagated to other
         *               nodes. This is set (for head node only) in
         *               doReleaseShared to ensure propagation
         *               continues, even if other operations have
         *               since intervened.
         *   0:          None of the above
         *
         * The values are arranged numerically to simplify use.
         * Non-negative values mean that a node doesn't need to
         * signal. So, most code doesn't need to check for particular
         * values, just for sign.
         *
         * The field is initialized to 0 for normal sync nodes, and
         * CONDITION for condition nodes.  It is modified using CAS
         * (or when possible, unconditional volatile writes).
         */
        volatile int waitStatus;

        /**
         * Link to predecessor node that current node/thread relies on
         * for checking waitStatus. Assigned during enqueuing, and nulled
         * out (for sake of GC) only upon dequeuing.  Also, upon
         * cancellation of a predecessor, we short-circuit while
         * finding a non-cancelled one, which will always exist
         * because the head node is never cancelled: A node becomes
         * head only as a result of successful acquire. A
         * cancelled thread never succeeds in acquiring, and a thread only
         * cancels itself, not any other node.
         */
        volatile Node prev;

        /**
         * Link to the successor node that the current node/thread
         * unparks upon release. Assigned during enqueuing, adjusted
         * when bypassing cancelled predecessors, and nulled out (for
         * sake of GC) when dequeued.  The enq operation does not
         * assign next field of a predecessor until after attachment,
         * so seeing a null next field does not necessarily mean that
         * node is at end of queue. However, if a next field appears
         * to be null, we can scan prev's from the tail to
         * double-check.  The next field of cancelled nodes is set to
         * point to the node itself instead of null, to make life
         * easier for isOnSyncQueue.
         */
        volatile Node next;

        /**
         * The thread that enqueued this node.  Initialized on
         * construction and nulled out after use.
         */
        volatile Thread thread;

        /**
         * Link to next node waiting on condition, or the special
         * value SHARED.  Because condition queues are accessed only
         * when holding in exclusive mode, we just need a simple
         * linked queue to hold nodes while they are waiting on
         * conditions. They are then transferred to the queue to
         * re-acquire. And because conditions can only be exclusive,
         * we save a field by using special value to indicate shared
         * mode.
         */
        Node nextWaiter;

        /**
         * Returns true if node is waiting in shared mode.
         */
        final boolean isShared() {
            return nextWaiter == SHARED;
        }

        /**
         * Returns previous node, or throws NullPointerException if null.
         * Use when predecessor cannot be null.  The null check could
         * be elided, but is present to help the VM.
         *
         * @return the predecessor of this node
         */
        final Node predecessor() throws NullPointerException {
            Node p = prev;
            if (p == null)
                throw new NullPointerException();
            else
                return p;
        }

        Node() {    // Used to establish initial head or SHARED marker
        }

        Node(Thread thread, Node mode) {     // Used by addWaiter
            this.nextWaiter = mode;
            this.thread = thread;
        }

        Node(Thread thread, int waitStatus) { // Used by Condition
            this.waitStatus = waitStatus;
            this.thread = thread;
        }
    }
```


    ReentrantLock 内部维护了一个Sync对象 ( Sync extends AbstractQueuedSynchronizer) 继承了抽象了aqs
    通过构造函数传递值来创建公平/非公平锁
    public ReentrantLock(boolean fair);
    fair 为true的时候创建一个公平锁  sync = new FairSync();
    fair 为false 的时候创建一个非公平锁  sync = new NonfairSync();
    
    然后分析一下公平锁/非公平锁是如何加锁和释放锁的呢？
    
### 加锁
    
    lock()  --> sync.lock(); 
            FairSync:
                ----FairSync.acquire(1)--->AQS.acuire(1)-->FairSync.tryAcuire(1)
    
            
            NonFairSync:
                ----NonfairSync.lock()-->compareAndSetStatus 获取锁
                                      -->acquire(1) 获取锁 --->AQS.acuire(1)-->NonfairSync.tryAcuire(1)-->Sync.nonfairTryAcqure


    如上面的流程图可以分析到公平和非公平锁的最大的区别在于2个地方
    1： FairSync 和 NonFairSync内部的lock方法
        FairSync的lock方法是直接调用acquire(1)方法
        NonfairSync的lock是做了一个判断： 
        如果compareAndSetState成功的那么直接设置当前线性为独占线程
        否则就进行acquire(1)

```java
static final class NonfairSync extends Sync {
    private static final long serialVersionUID = 7316153563782823691L;

    /**
     * Performs lock.  Try immediate barge, backing up to normal
     * acquire on failure.
     */
    final void lock() {
        if (compareAndSetState(0, 1))
            setExclusiveOwnerThread(Thread.currentThread());
        else
            acquire(1);
    }

    protected final boolean tryAcquire(int acquires) {
        return nonfairTryAcquire(acquires);
    }
    
    /**
    final boolean nonfairTryAcquire(int acquires) {
        final Thread current = Thread.currentThread();
        int c = getState();
        if (c == 0) {
            if (compareAndSetState(0, acquires)) {
                setExclusiveOwnerThread(current);
                return true;
            }
        }
        else if (current == getExclusiveOwnerThread()) {
            int nextc = c + acquires;
            if (nextc < 0) // overflow
                throw new Error("Maximum lock count exceeded");
            setState(nextc);
            return true;
        }
        return false;
    }*/
}
```    

    2: tryAcquire方法不同
        FairSync的tryAcquire 方法说先判断state如果是0 那么进行判断是否当前线程所在的节点前面是否有等待的（其实就是排队的获取锁）
        并且cas state 成功就获取到锁， 如果当前的线程已经获取到了锁那么state ++ ;
        
        
        NonfairSync 是调用的tryAcquire-->nonfairTryAcquire   
        直接通过compareAndSetState来获取锁 不考虑当前的节点之前是否有等待的节点了  （可以理解为插队获取锁）

```java
static final class FairSync extends Sync {
    private static final long serialVersionUID = -3000897897090466540L;

    final void lock() {
        acquire(1);
    }

    /**
     * Fair version of tryAcquire.  Don't grant access unless
     * recursive call or no waiters or is first.
     */
    protected final boolean tryAcquire(int acquires) {
        final Thread current = Thread.currentThread();
        int c = getState();
        if (c == 0) {
            if (!hasQueuedPredecessors() &&
                compareAndSetState(0, acquires)) {
                setExclusiveOwnerThread(current);
                return true;
            }
        }
        else if (current == getExclusiveOwnerThread()) {
            int nextc = c + acquires;
            if (nextc < 0)
                throw new Error("Maximum lock count exceeded");
            setState(nextc);
            return true;
        }
        return false;
    }
}
```    


### 释放锁
    unlock -->AQS.release()-->Sync.tryRelease()
    所以是否锁的动作都是一样的
    是否锁的是如何操作的呢？
    

```java
class AbstractQueuedSynchronizer{
   private void unparkSuccessor(Node node) {
       /*
        * If status is negative (i.e., possibly needing signal) try
        * to clear in anticipation of signalling.  It is OK if this
        * fails or if status is changed by waiting thread.
        */
       int ws = node.waitStatus;
       if (ws < 0)
           //当前的节点修改状态为0
           compareAndSetWaitStatus(node, ws, 0);

       /*
        * Thread to unpark is held in successor, which is normally
        * just the next node.  But if cancelled or apparently null,
        * traverse backwards from tail to find the actual
        * non-cancelled successor.
        */
       //获取当前节点的下一个节点
       Node s = node.next;
       if (s == null || s.waitStatus > 0) {
           s = null;
           for (Node t = tail; t != null && t != node; t = t.prev)
               if (t.waitStatus <= 0)
                   s = t;
       }
       if (s != null)
           //如果下一个节点不为空， 那么通过LockSupport方法将下个节点的线程唤醒
           LockSupport.unpark(s.thread);
   } 
}
```   


###总结
    ReentrantLock内部实现了公平锁和非公平锁； 
    公平锁好比是一个排队的有序的队列 一个一个的执行
    非公平锁好比是一个队列，总是有那么几个不守规矩的人插队， 打破了有序的规则；
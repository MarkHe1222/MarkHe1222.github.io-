---
title: Java AQS详解
date: 2021-07-15 00:20:14
categories: 语言
tags: Java
---

AbstractQueuedSynchroizer简称AQS，是一个抽象的队列式同步框架，提供了阻塞锁和 FIFO 队列实现同步操作，用来构建锁和同步器。JUC 包中的同步类基本都是基于 AQS 同步器来实现的，如 ReentrantLock，Semaphore 等。

AQS采用了模板方法设计模式，支持通过子类重写相应的方法实现不同的同步器。在AQS中，有一个state变量，表示同步状态(同步状态可以看作是一种资源，对同步状态的获取可以看作是对同步资源的竞争)，AQS提供了多种获取同步状态的方式，包括独占式获取、共享式获取以及超时获取等。

<!-- more -->
<!-- markdownlint-disable MD041 MD002--> 

## 1 核心思想

假设，有四个线程由于业务需求需要同时占用某资源，但该资源在同一个时刻只能被其中唯一线程所独占。那么此时应该如何标识该资源已经被独占，同时剩余无法获取该资源的线程应该如何竞争？

![AQS](Java-AQS%E8%AF%A6%E8%A7%A3/AQS.png)

这里就涉及到了关于共享资源的竞争与同步关系。对于不同的场景来说，实现的思路可能会有不同。AQS 正是为了解决这个问题而被设计出来的。AQS 是一个集同步状态管理、线程阻塞、线程释放及队列管理功能于一身的同步框架。其核心思想是当多个线程竞争资源时会将未成功竞争到资源的线程构造为 Node 节点放置到一个双向 FIFO 队列中。被放入到该队列中的线程会保持阻塞直至被前驱节点唤醒。值得注意的是该队列中只有队首节点有资格被唤醒竞争锁。

AQS使用一个int成员变量state来表示同步状态，通过内置的FIFO队列来完成获取资源线程的排队工作。AQS使用CAS对该同步状态进行原子操作实现对其值的修改。

```java
    /**
     * The synchronization state.
     */
		//共享变量，使用volatile修饰保证线程可见性
    private volatile int state;
```

状态变量state通过procted类型的getState，setState，compareAndSetState方法操作，改变同步状态。

```java
    /**
     * Returns the current value of synchronization state.
     * This operation has memory semantics of a {@code volatile} read.
     * @return current state value
     */
		//返回同步状态的当前值
    protected final int getState() {
        return state;
    }

    /**
     * Sets the value of synchronization state.
     * This operation has memory semantics of a {@code volatile} write.
     * @param newState the new state value
     */
		 // 设置同步状态的值
    protected final void setState(int newState) {
        state = newState;
    }

    /**
     * Atomically sets synchronization state to the given updated
     * value if the current state value equals the expected value.
     * This operation has memory semantics of a {@code volatile} read
     * and write.
     *
     * @param expect the expected value
     * @param update the new value
     * @return {@code true} if successful. False return indicates that the actual
     *         value was not equal to the expected value.
     */
		//原子地(CAS操作)将同步状态值设置为给定值update, 如果当前同步状态的值等于expect(期望值)
    protected final boolean compareAndSetState(int expect, int update) {
        // See below for intrinsics setup to support this
        return unsafe.compareAndSwapInt(this, stateOffset, expect, update);
    }
```

## 2 CLH锁队列

同步队列被称为CLH队列，是Craig，Landin，Hagersten的合称。AQS通过内置的FIFO同步双向队列来完成资源获取线程的排队工作，内部通过节点head`实际上是虚拟节点，真正的第一个线程在head.next的位置`和tail记录队首和队尾元素，队列元素类型为Node。

![CLH](Java-AQS%E8%AF%A6%E8%A7%A3/CLH.png)

CLH是虚拟的双向队列，底层是双向链表，包括head节点和tail结点，仅存在结点之间的关联关系。AQS将每条请求共享资源的线程封装成一个CLH锁队列的一个结点(Node)来实现锁的分配。

- 如果当前线程获取同步状态失败（锁）时，AQS 则会将当前线程以及等待状态等信息构造成一个节点（Node）并将其加入同步队列，同时会阻塞当前线程
- 当同步状态释放时，则会把节点中的线程唤醒，使其再次尝试获取同步状态。

#### Node 数据结构分析

```java
  // 结点的数据结构  
	static final class Node {
        /** Marker to indicate a node is waiting in shared mode */
        // 表示该节点等待模式为共享式，通常记录于nextWaiter，
    		// 通过判断nextWaiter的值可以判断当前结点是否处于共享模式
        static final Node SHARED = new Node();
    
        /** Marker to indicate a node is waiting in exclusive mode */
        // 表示节点处于独占式模式，与SHARED相对
        static final Node EXCLUSIVE = null;

     		//waitStatus的不同状态，具体内容见下文的表格
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
        volatile int waitStatus;

	      // 记录前置结点
        volatile Node prev;
				// 记录后置结点
        volatile Node next;
				// 记录当前的线程
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
    		// 用于记录共享模式(SHARED), 也可以用来记录CONDITION队列
        Node nextWaiter;
    
				// 通过nextWaiter的记录值判断当前结点的模式是否为共享模式
        final boolean isShared() {
            return nextWaiter == SHARED;
        }

				// 获取当前结点的前置结点
        final Node predecessor() throws NullPointerException {
            Node p = prev;
            if (p == null)
                throw new NullPointerException();
            else
                return p;
        }

    		// 用于初始化时创建head结点或者创建SHARED结点
        Node() {    
        }
				// 在addWaiter方法中使用，用于创建一个新的结点
        Node(Thread thread, Node mode) {     // Used by addWaiter
            this.nextWaiter = mode;
            this.thread = thread;
        }
				// 在CONDITION队列中使用该构造函数新建结点
        Node(Thread thread, int waitStatus) { // Used by Condition
            this.waitStatus = waitStatus;
            this.thread = thread;
        }
    }

    /**
     * Head of the wait queue, lazily initialized.  Except for
     * initialization, it is modified only via method setHead.  Note:
     * If head exists, its waitStatus is guaranteed not to be
     * CANCELLED.
     */
		// 记录头结点
    private transient volatile Node head;

    /**
     * Tail of the wait queue, lazily initialized.  Modified only via
     * method enq to add new wait node.
     */
		// 记录尾结点
    private transient volatile Node tail;
```

#### Node 的状态表「waitStatus」

| 状态名    | 状态值 | 描述                                                         |
| :-------- | :----: | :----------------------------------------------------------- |
| INITAL    |   0    | 节点初始化默认值                                             |
| CANCELLED |   1    | 取消状态，如果当前线程的前置节点状态为 CANCELLED，则表明前置节点已经等待超时或者已经被中断了，这时需要将其从等待队列中删除 |
| SIGNAL    |   -1   | 等待触发状态，如果当前线程的**前置**节点状态为 SIGNAL，则表明当前线程需要阻塞 |
| CONDITION |   -2   | 等待条件状态，表示当前节点在等待 condition，即在 condition 队列中。当其他线程对Condtion调用了signal方法后，该节点将会从等待队列中转移到同步队列中，加入到对同步状态的获取中 |
| PROPAGATE |   -3   | 状态需要向后传播，表示 releaseShared 需要被传播给后续节点，仅在共享锁模式下使用 |

## 3 AQS对资源的共享方式

线程同步的关键是对state进行操作，根据state是否属于一个线程，操作state的方式有两种模式。
**a. 独占模式「Exclusive」**：只有一个线程能执行。使用独占的方式获取的资源是与具体线程绑定的，如果一个线程获取到了资源，便标记这个线程已经获取到，其他线程再次尝试操作state获取资源时就会发现当前该资源不是自己持有的，就会在获取失败后阻塞。又可分为公平锁和非公平锁：

- 公平锁：按照线程在队列中的排队顺序获取锁；
- 非公平锁：当线程要获取锁时，无视队列顺序直接去抢锁，谁抢到就是谁的，所以非公平锁效率较高；

**b 共享模式「 Share 」**：多个线程可同时执行。对应共享方式的资源与具体线程是不相关的，当多个线程去请求资源时通过CAS 方式竞争获取资源，当一个线程获取到了资源后，另外一个线程再次去获取时如果当前资源还能满足它的需要，则当前线程只需要使用CAS 方式进行获取即可。

## 4 AQS的设计模式

AQS 同步器的设计是基于模板方法模式。使用者继承AbstractQueuedSynchronizer并重写指定的方法。实现对于共享资源state的获取和释放。

| 方法                      | 作用                                                         |
| :------------------------ | :----------------------------------------------------------- |
| tryAcquire(int arg)       | **独占模式**尝试获取资源。实现该方法需要查询当前状态是否符合预期，然后进行相应的状态更新实现控制(获取成功返回true，否则返回false，成功通常是可以更新同步状态，失败则是不符合更新同步状态的条件)，其中arg表示需要获取的同步状态数 |
| tryRelease(int arg)       | **独占模式**尝试释放资源。同时更新同步状态(通常在同步状态state更新为0才会返回true，表示已经彻底释放同步资源)，其中arg表示需要释放的同步状态数 |
| tryAcquireShared(int arg) | **共享式**获取同步状态，同时更新同步状态。负数表示失败；0表示成功，但没有剩余可用资源；正数表示成功，且有剩余资源。 |
| tryReleaseShared(int arg) | **共享式**释放同步状态，同时更新同步状态。                   |
| isHeldExclusively()       | 一般用于判断同步器是否被当前线程独占，只有用到condition才需要去实现它。 |

## 5 继承的父类

AbstractQueuedSynchronizer继承自AbstractOwnableSynchronizer抽象类，并且实现了Serializable接口，可以进行序列化。

AbstractOwnableSynchronizer源码分析

```java
public abstract class AbstractOwnableSynchronizer
    implements java.io.Serializable {

    /** Use serial ID even though all fields transient. */
  	// 版本序列号
    private static final long serialVersionUID = 3737899427754241961L;

    /**
     * Empty constructor for use by subclasses.
     */
    protected AbstractOwnableSynchronizer() { }

    /**
     * The current owner of exclusive mode synchronization.
     */
  	// 独占模式下的线程
    private transient Thread exclusiveOwnerThread;

    /**
     * Sets the thread that currently owns exclusive access.
     * A {@code null} argument indicates that no thread owns access.
     * This method does not otherwise impose any synchronization or
     * {@code volatile} field accesses.
     * @param thread the owner thread
     */
  	// 设置独占线程 
    protected final void setExclusiveOwnerThread(Thread thread) {
        exclusiveOwnerThread = thread;
    }

    /**
     * Returns the thread last set by {@code setExclusiveOwnerThread},
     * or {@code null} if never set.  This method does not otherwise
     * impose any synchronization or {@code volatile} field accesses.
     * @return the owner thread
     */
  	// 获取独占线程 
    protected final Thread getExclusiveOwnerThread() {
        return exclusiveOwnerThread;
    }
}
```

## 6 核心方法介绍

此文档中，先介绍AQS中核心的方法，在使用中，进行串联。

#### acquire

```java
    /**
     * Acquires in exclusive mode, ignoring interrupts.  Implemented
     * by invoking at least once {@link #tryAcquire},
     * returning on success.  Otherwise the thread is queued, possibly
     * repeatedly blocking and unblocking, invoking {@link
     * #tryAcquire} until success.  This method can be used
     * to implement method {@link Lock#lock}.
     *
     * @param arg the acquire argument.  This value is conveyed to
     *        {@link #tryAcquire} but is otherwise uninterpreted and
     *        can represent anything you like.
     */
    public final void acquire(int arg) {
        // 如果线程直接获取成功，或者再尝试获取成功后都是直接工作，
		    // 如果是从阻塞状态中唤醒开始工作的线程，将当前的线程中断
        if (!tryAcquire(arg) &&
            acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
            selfInterrupt();
    }
```

当一个线程调用acquire时，调用方法流程如下

![raftuserstudy2013](Java-AQS%E8%AF%A6%E8%A7%A3/raftuserstudy2013.png)

1. 首先调用tryAcquire方法，调用此方法的线程会试图在独占模式下获取对象状态。此方法应该查询是否允许它在独占模式下获取对象状态，如果允许，则获取它。在AbstractQueuedSynchronizer源码中默认会抛出一个异常，即需要子类去重写此方法完成自己的逻辑。
2. 若tryAcquire失败，则调用addWaiter方法，addWaiter方法完成的功能是将调用此方法的线程封装成为一个结点并放入CLH队列中。
3. 调用acquireQueued方法，此方法完成的功能是CLH中的结点不断尝试获取资源，若成功，则返回true，否则，返回false。

#### tryAcquire - override

```java
// 需要子类去重写此方法完成自己的逻辑 - 试图在独占模式下获取对象状态
protected boolean tryAcquire(int arg) {
    throw new UnsupportedOperationException();
}
```

#### addWaiter

```java
    /**
     * Creates and enqueues node for current thread and given mode.
     *
     * @param mode Node.EXCLUSIVE for exclusive, Node.SHARED for shared
     * @return the new node
     */
		// 添加等待者
    private Node addWaiter(Node mode) {
      	// 新生成一个结点，默认为独占模式 addWaiter(Node.EXCLUSIVE)
        Node node = new Node(Thread.currentThread(), mode);
        // 保存尾结点 
        Node pred = tail;
        if (pred != null) { // 尾结点不为空，队列已经有节点在等待
            node.prev = pred; // 将node结点的prev域连接到尾结点
            if (compareAndSetTail(pred, node)) {  // 比较pred是否为尾结点，是-则将尾结点设置为node 
                pred.next = node; // 设置尾结点的next域为node
                return node; // 返回新生成的结点
            }
        }
        enq(node); // 第一次往队列中新增节点时，会执行enq方法，或者是compareAndSetTail操作失败，则入队列
        return node;
    }
```

#### enq

```JAVA
    /**
     * Inserts node into queue, initializing if necessary. See picture above.
     * @param node the node to insert
     * @return node's predecessor
     */
		// 将节点插入阻塞队列 - 尾结点
    private Node enq(final Node node) {
        for (;;) { // 无线循环，确保节点的成功插入
            Node t = tail; // 保存尾结点
            if (t == null) { // 尾结点为空，即还没被初始化
                if (compareAndSetHead(new Node())) // 头结点为空，并设置头结点为新生成的结点	
                    tail = head; // 头结点与尾结点都指向同一个新生结点
            } else { // 尾结点不为空，即已经被初始化过
                node.prev = t; // 将node结点的prev域连接到尾结点
                if (compareAndSetTail(t, node)) { // 比较结点t是否为尾结点，是-则将尾结点设置为node
                    t.next = node; // 设置node为新的尾结点
                    return t; // 返回原-尾结点
                }
            }
        }
    }
```

#### acquireQueued

首先获取当前节点的前驱节点，如果前驱节点是头结点并且能够获取(资源)，代表该当前节点能够占有锁，设置头结点为当前节点，返回。

```java
    /**
     * Acquires in exclusive uninterruptible mode for thread already in
     * queue. Used by condition wait methods as well as acquire.
     *
     * @param node the node
     * @param arg the acquire argument
     * @return {@code true} if interrupted while waiting
     */
		// 以独占不间断模式获取已在队列中的阻塞线程。
    final boolean acquireQueued(final Node node, int arg) {
        boolean failed = true; // 标志
        try {
            boolean interrupted = false; // 中断标志
            for (;;) { // 无限循环
                final Node p = node.predecessor(); // 获取node节点的前驱结点
                if (p == head && tryAcquire(arg)) { // 前驱为头结点并且成功获得锁
                    setHead(node); // 设置头结点
                    p.next = null; // help GC
                    failed = false; // 设置标志
                    return interrupted;
                }
                if (shouldParkAfterFailedAcquire(p, node)&&//当获取(资源)失败后，检查并且更新结点状态
                    parkAndCheckInterrupt()) //执行park操作
                    interrupted = true;
            }
        } finally {
            if (failed)
                cancelAcquire(node);
        }
    }
```

acquireQueued方法的整个的逻辑：

1. 判断结点的前驱是否为head并且当前线程是否再次成功获取(资源)。
2. 若步骤1均满足，则设置结点为head，之后会判断是否finally模块，然后返回。
3. 若步骤2不满足，则判断是否需要park当前线程，是否需要park当前线程的逻辑是判断结点的前驱结点的状态是否为SIGNAL，若是，则park当前结点，否则，不进行park操作。
4. 若park了当前线程，之后某个线程对本线程unpark后，并且本线程也获得机会运行。那么，将会继续进行步骤1的判断。

#### shouldParkAfterFailedAcquire

只有当该节点的前驱结点的状态为SIGNAL时，才可以对该结点所封装的线程进行park操作。否则，将不能进行park操作。

```java
    /**
     * Checks and updates status for a node that failed to acquire.
     * Returns true if thread should block. This is the main signal
     * control in all acquire loops.  Requires that pred == node.prev.
     *
     * @param pred node's predecessor holding status
     * @param node the node
     * @return {@code true} if thread should block
     */
		// 当获取(资源)失败后，检查并且更新结点状态
    private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
        int ws = pred.waitStatus; // 获取前驱结点的状态
        if (ws == Node.SIGNAL) // 如果前驱结点的状态为SIGNAL，那么当前的结点应该阻塞（SIGNAL定义）
            /*
             * This node has already set status asking a release
             * to signal it, so it can safely park.
             */
            return true; // 可以进行park操作
        if (ws > 0) { // 如果前驱结点已经为取消状态（CANCELLED 1 > 0）则向前遍历，直到找到一个有效的结点
            /*
             * Predecessor was cancelled. Skip over predecessors and
             * indicate retry.
             */
            do {
                node.prev = pred = pred.prev;
            } while (pred.waitStatus > 0); // 找到pred结点前面最近的一个状态不为CANCELLED的结点
            pred.next = node; // 将当前结点与新找到的前驱结点连接
        } else {
            /*
             * waitStatus must be 0 or PROPAGATE.  Indicate that we
             * need a signal, but don't park yet.  Caller will need to
             * retry to make sure it cannot acquire before parking.
             */
         	 	// 到这里的结点状态可能为：CONDITION 和 PROPAGATE，比较并设置前驱结点的状态为SIGNAL
            compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
        }
        return false; // 不能进行park操作
    }
```

####  parkAndCheckInterrupt

parkAndCheckInterrupt方法里的逻辑是首先执行park操作，即禁用当前线程，然后返回该线程是否已经被中断。

```java
    /**
     * Convenience method to park and then check if interrupted
     *
     * @return {@code true} if interrupted
     */
		// 进行park操作并且返回该线程是否被中断
    private final boolean parkAndCheckInterrupt() {
      	// 在许可可用之前禁用当前线程，并且设置了blocker
        LockSupport.park(this);
        return Thread.interrupted();  // 当前线程是否已被中断，并清除中断标记位
    }
```

#### cancelAcquire

该方法完成的功能就是取消当前线程对资源的获取，即设置该结点的状态为CANCELLED

```java
    /**
     * Cancels an ongoing attempt to acquire.
     *
     * @param node the node
     */
		// 取消线程继续获取资源
    private void cancelAcquire(Node node) {
        // Ignore if node doesn't exist
        if (node == null) // node为空，返回
            return;

        node.thread = null; // 设置node结点的thread为空

        // Skip cancelled predecessors
        Node pred = node.prev; // 保存node的前驱结点
        while (pred.waitStatus > 0) //若前驱结点为CANCELLED转态，继续向前遍历，找到第一个状态小于0的结点
            node.prev = pred = pred.prev;

        // predNext is the apparent node to unsplice. CASes below will
        // fail if not, in which case, we lost race vs another cancel
        // or signal, so no further action is necessary.
        Node predNext = pred.next;// 获取pred结点的下一个结点

        // Can use unconditional write instead of CAS here.
        // After this atomic step, other Nodes can skip past us.
        // Before, we are free of interference from other threads.
        node.waitStatus = Node.CANCELLED;// 设置当前node结点的状态为CANCELLED

        // If we are the tail, remove ourselves.
        if (node == tail && compareAndSetTail(node, pred)) { //如果node结点为尾结点，则设置尾结点为pred结点
            compareAndSetNext(pred, predNext, null); // 比较并设置pred结点的next节点为null
        } else { // node结点不为尾结点，或者比较设置新的尾结点不成功
            // If successor needs signal, try to set pred's next-link
            // so it will get one. Otherwise wake it up to propagate.
            int ws;
            if (pred != head &&
                ((ws = pred.waitStatus) == Node.SIGNAL || // pred结点不为头结点，并且pred结点的状态为SIGNAL
                 (ws <= 0 && compareAndSetWaitStatus(pred, ws, Node.SIGNAL))) && // 或者pred结点状态小于等于0，并且比较并设置等待状态为SIGNAL成功
                pred.thread != null) { // 并且pred结点所封装的线程不为空
                Node next = node.next; // 保存node结点的后继结点
                if (next != null && next.waitStatus <= 0) // 后继不为空并且后继的状态小于等于0
                    compareAndSetNext(pred, predNext, next); // 比较并设置pred.next = next;
            } else {
                unparkSuccessor(node); // 唤醒当前node结点的后继结点
            }

            node.next = node; // help GC
        }
    }
```

#### unparkSuccessor

该方法的作用就是为了释放node节点的后继结点。

```java
    /**
     * Wakes up node's successor, if one exists.
     *
     * @param node the node
     */
	 	// 唤醒当前node结点的后继结点
    private void unparkSuccessor(Node node) {
        /*
         * If status is negative (i.e., possibly needing signal) try
         * to clear in anticipation of signalling.  It is OK if this
         * fails or if status is changed by waiting thread.
         */
        int ws = node.waitStatus; // 获取node结点的等待状态
        if (ws < 0)  // 状态值小于0，为SIGNAL -1 或 CONDITION -2 或 PROPAGATE -3
            compareAndSetWaitStatus(node, ws, 0);  // 比较并且设置结点等待状态，设置为INITAL 0 

        /*
         * Thread to unpark is held in successor, which is normally
         * just the next node.  But if cancelled or apparently null,
         * traverse backwards from tail to find the actual
         * non-cancelled successor.
         */
        Node s = node.next; // 获取node的后继结点
        if (s == null || s.waitStatus > 0) { //如果node的后继结点为空，或者结点等待状态CANCELLED 1
            s = null; // s赋值为空
            for (Node t = tail; t != null && t != node; t = t.prev) // 从尾结点开始从后往前开始遍历
                if (t.waitStatus <= 0)// 找到等待状态小于等于0的结点，找到最前的状态小于等于0的结点
                    s = t; // 保存结点
        }
        if (s != null) // 该结点不为为空，释放许可
            LockSupport.unpark(s.thread);
    }
```

#### release

以独占模式释放对象

```java
    /**
     * Releases in exclusive mode.  Implemented by unblocking one or
     * more threads if {@link #tryRelease} returns true.
     * This method can be used to implement method {@link Lock#unlock}.
     *
     * @param arg the release argument.  This value is conveyed to
     *        {@link #tryRelease} but is otherwise uninterpreted and
     *        can represent anything you like.
     * @return the value returned from {@link #tryRelease}
     */
    public final boolean release(int arg) {
        if (tryRelease(arg)) { // 释放成功(自定义实现)
            Node h = head; // 保存头结点
            if (h != null && h.waitStatus != 0) // 头结点不为空并且头结点状态不为0
                unparkSuccessor(h); //释放头结点的后继结点
            return true;
        }
        return false;
    }
```

#### tryRelease - override

tryRelease的默认实现是抛出异常，需要具体的子类实现，如果tryRelease成功，那么如果头结点不为空并且头结点的状态不为INITAL 0，则释放头结点的后继结点

```java
    /**
     * Attempts to set the state to reflect a release in exclusive
     * mode.
     *
     * <p>This method is always invoked by the thread performing release.
     *
     * <p>The default implementation throws
     * {@link UnsupportedOperationException}.
     *
     * @param arg the release argument. This value is always the one
     *        passed to a release method, or the current state value upon
     *        entry to a condition wait.  The value is otherwise
     *        uninterpreted and can represent anything you like.
     * @return {@code true} if this object is now in a fully released
     *         state, so that any waiting threads may attempt to acquire;
     *         and {@code false} otherwise.
     * @throws IllegalMonitorStateException if releasing would place this
     *         synchronizer in an illegal state. This exception must be
     *         thrown in a consistent fashion for synchronization to work
     *         correctly.
     * @throws UnsupportedOperationException if exclusive mode is not supported
     */
    protected boolean tryRelease(int arg) {
        throw new UnsupportedOperationException();
    }
```

#### transferForSignal

将结点从Condition队列转移到Sync队列

```java
    /**
     * Transfers a node from a condition queue onto sync queue.
     * Returns true if successful.
     * @param node the node
     * @return true if successfully transferred (else the node was
     * cancelled before signal)
     */
    final boolean transferForSignal(Node node) {
        /*
         * If cannot change waitStatus, the node has been cancelled.
         */
        if (!compareAndSetWaitStatus(node, Node.CONDITION, 0))//将结点从CONDITION状态改为初始化状态
            return false;

        /*
         * Splice onto queue and try to set waitStatus of predecessor to
         * indicate that thread is (probably) waiting. If cancelled or
         * attempt to set waitStatus fails, wake up to resync (in which
         * case the waitStatus can be transiently and harmlessly wrong).
         */
        Node p = enq(node); // 结点状态改为0，并且加入资源队列，p为node的前置结点
        int ws = p.waitStatus; // 获取p的状态
        if (ws > 0 || !compareAndSetWaitStatus(p, ws, Node.SIGNAL)) // 如果前置结点为CANCELLED状态 或者 状态设置为SIGNAL失败
            LockSupport.unpark(node.thread); // 将node结点解锁
        return true;
    }
```

#### fullyRelease

使用当前状态值调用 release； 返回保存状态。 取消节点并在失败时抛出异常。

```java
    /**
     * Invokes release with current state value; returns saved state.
     * Cancels node and throws exception on failure.
     * @param node the condition node for this wait
     * @return previous sync state
     */
		// 将当前的结点以独占模式释放资源，并且保留当前状态
    final int fullyRelease(Node node) {
        boolean failed = true; // 保存释放是否成功的状态
        try {
            int savedState = getState(); // 获取当前结点状态
            if (release(savedState)) { // 以独占模式释放资源
                failed = false; // 结点释放成功
                return savedState; // 返回同步状态
            } else {
                throw new IllegalMonitorStateException(); // 结点释放失败，抛出异常
            }
        } finally {
            if (failed) // 释放失败
                node.waitStatus = Node.CANCELLED; // 将结点状态标记为 CANCELLED
        }
    }
```

#### isOnSyncQueue

如果一个结点（始终是最初放置在Condition队列中的结点）现在正在等待重新获取同步队列，则返回 true。

```java
    /**
     * Returns true if a node, always one that was initially placed on
     * a condition queue, is now waiting to reacquire on sync queue.
     * @param node the node
     * @return true if is reacquiring
     */
    final boolean isOnSyncQueue(Node node) {
        if (node.waitStatus == Node.CONDITION || node.prev == null) // 如果状态是CONDITION 或者 结点是队列第一个结点 (不在CLH队列)
            return false; 
     		//如果当前节点有next指针（next指针只在CLH队列中的节点有，条件队列中的节点是nextWaiter）的话，就返回true (Condition队列中 下个结点是用 nextWaiter)
        if (node.next != null) // If has successor, it must be on queue
            return true; 
        /*
         * node.prev can be non-null, but not yet on queue because
         * the CAS to place it on queue can fail. So we have to
         * traverse from tail to make sure it actually made it.  It
         * will always be near the tail in calls to this method, and
         * unless the CAS failed (which is unlikely), it will be
         * there, so we hardly ever traverse much.
         */
        return findNodeFromTail(node); //如果上面无法快速判断的话，就只能从CLH队列中进行遍历，一个一个地去进行判断了
    }
```

#### findNodeFromTail

从CLH队列的最后结点向前遍历，依次判断

```java
    /**
     * Returns true if node is on sync queue by searching backwards from tail.
     * Called only when needed by isOnSyncQueue.
     * @return true if present
     */
    private boolean findNodeFromTail(Node node) {
        Node t = tail;
        for (;;) {
            if (t == node)
                return true;
            if (t == null)
                return false;
            t = t.prev;
        }
    }
```

#### transferAfterCancelledWait

如有必要，在取消等待后将节点传输到同步队列。如果线程在发出信号之前被取消，则返回true。

```java
    /**
     * Transfers node, if necessary, to sync queue after a cancelled wait.
     * Returns true if thread was cancelled before being signalled.
     *
     * @param node the node
     * @return true if cancelled before the node was signalled
     */
    final boolean transferAfterCancelledWait(Node node) {
	      // 尝试使用CAS操作将node(CONDITION状态) 的ws设置为0
        if (compareAndSetWaitStatus(node, Node.CONDITION, 0)) { 
            enq(node); // 设置成功后，进入CLH队列
            return true; // 返回
        }
        /*
         * If we lost out to a signal(), then we can't proceed
         * until it finishes its enq().  Cancelling during an
         * incomplete transfer is both rare and transient, so just
         * spin.
         */
        while (!isOnSyncQueue(node)) // 判断节点是否在CLH队列中（不在队列中）
            Thread.yield(); 
        return false; // 返回
    }
```



## 7 扩展

### Condition接口

Contition是一种广义上的条件队列，它利用await()和signal()为线程提供了一种**更为灵活的等待/通知模式**。

Condition必须要配合Lock一起使用，因为对共享状态变量的访问发生在多线程环境下。一个Condition的实例必须与一个Lock绑定，因此await和signal的调用必须在lock和unlock之间，有锁之后，才能使用condition。

```java
public interface Condition {
	  // 使当前线程等待，直到它收到信号或被中断
    void await() throws InterruptedException;
  	// 使当前线程等待，直到它收到信号
    void awaitUninterruptibly();
  	// 使当前线程等待，直到它被发出信号或被中断，或者指定的等待时间过去
    long awaitNanos(long nanosTimeout) throws InterruptedException;
  	// 使当前线程等待，直到它被发出信号或被中断，或者指定的等待时间过去
    boolean await(long time, TimeUnit unit) throws InterruptedException;
  	// 使当前线程等待，直到它被发出信号或被中断，或者指定的截止日期过去
    boolean awaitUntil(Date deadline) throws InterruptedException;
  	// 唤醒一个等待线程
    void signal();
  	// 唤醒所有等待线程
    void signalAll();
}
```

### ConditionObject内部类

ConditionObject是AQS的内部类，实现了Condition接口，Lock中提供newCondition()方法，委托给内部AQS的实现Sync来创建ConditionObject对象，使用AQS中定义的Condition。

ConditionObject用来结合锁实现线程同步，**ConditionObject可以直接访问AQS对象内部的变量，比如state状态值和AQS队列**。ConditionObject是条件变量，每个条件变量对应一个**条件队列**（单向链表队列），用来存放调用条件变量的await方法后被阻塞的线程。

需要明确这里的Condition队列和CLH同步队列是不一样的：

- AQS维护的是当前在等待资源的队列，Condition维护的是在等待signal信号的队列；
- 每个线程会存在上述两个队列中的一个，lock与unlock对应在AQS队列，signal与await对应条件队列，线程节点在他们之间进行切换；

#### 构造方法 + 属性

```java
    public class ConditionObject implements Condition, java.io.Serializable {
      	// 序列化版本号
        private static final long serialVersionUID = 1173984872572414699L;
        /** First node of condition queue. */
	      // condition队列的头结点
        private transient Node firstWaiter;
        /** Last node of condition queue. */
      	// condition队列的尾结点
        private transient Node lastWaiter;

				// 构造方法
        public ConditionObject() { }

        /*
         * For interruptible waits, we need to track whether to throw
         * InterruptedException, if interrupted while blocked on
         * condition, versus reinterrupt current thread, if
         * interrupted while blocked waiting to re-acquire.
         */

        /** Mode meaning to reinterrupt on exit from wait */
        private static final int REINTERRUPT =  1;
        /** Mode meaning to throw InterruptedException on exit from wait */
        private static final int THROW_IE    = -1;

        /**
         * Checks for interrupt, returning THROW_IE if interrupted
         * before signalled, REINTERRUPT if after signalled, or
         * 0 if not interrupted.
         */
        private int checkInterruptWhileWaiting(Node node) {
            return Thread.interrupted() ?
                (transferAfterCancelledWait(node) ? THROW_IE : REINTERRUPT) :
                0;
        }

        /**
         * Throws InterruptedException, reinterrupts current thread, or
         * does nothing, depending on mode.
         */
        private void reportInterruptAfterWait(int interruptMode)
            throws InterruptedException {
            if (interruptMode == THROW_IE)
                throw new InterruptedException();
            else if (interruptMode == REINTERRUPT)
                selfInterrupt();
        }

        /**
         * Implements absolute timed condition wait.
         * <ol>
         * <li> If current thread is interrupted, throw InterruptedException.
         * <li> Save lock state returned by {@link #getState}.
         * <li> Invoke {@link #release} with saved state as argument,
         *      throwing IllegalMonitorStateException if it fails.
         * <li> Block until signalled, interrupted, or timed out.
         * <li> Reacquire by invoking specialized version of
         *      {@link #acquire} with saved state as argument.
         * <li> If interrupted while blocked in step 4, throw InterruptedException.
         * <li> If timed out while blocked in step 4, return false, else true.
         * </ol>
         */
        public final boolean awaitUntil(Date deadline)
                throws InterruptedException {
            long abstime = deadline.getTime();
            if (Thread.interrupted())
                throw new InterruptedException();
            Node node = addConditionWaiter();
            int savedState = fullyRelease(node);
            boolean timedout = false;
            int interruptMode = 0;
            while (!isOnSyncQueue(node)) {
                if (System.currentTimeMillis() > abstime) {
                    timedout = transferAfterCancelledWait(node);
                    break;
                }
                LockSupport.parkUntil(this, abstime);
                if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
                    break;
            }
            if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
                interruptMode = REINTERRUPT;
            if (node.nextWaiter != null)
                unlinkCancelledWaiters();
            if (interruptMode != 0)
                reportInterruptAfterWait(interruptMode);
            return !timedout;
        }

        /**
         * Implements timed condition wait.
         * <ol>
         * <li> If current thread is interrupted, throw InterruptedException.
         * <li> Save lock state returned by {@link #getState}.
         * <li> Invoke {@link #release} with saved state as argument,
         *      throwing IllegalMonitorStateException if it fails.
         * <li> Block until signalled, interrupted, or timed out.
         * <li> Reacquire by invoking specialized version of
         *      {@link #acquire} with saved state as argument.
         * <li> If interrupted while blocked in step 4, throw InterruptedException.
         * <li> If timed out while blocked in step 4, return false, else true.
         * </ol>
         */
        public final boolean await(long time, TimeUnit unit)
                throws InterruptedException {
            long nanosTimeout = unit.toNanos(time);
            if (Thread.interrupted())
                throw new InterruptedException();
            Node node = addConditionWaiter();
            int savedState = fullyRelease(node);
            final long deadline = System.nanoTime() + nanosTimeout;
            boolean timedout = false;
            int interruptMode = 0;
            while (!isOnSyncQueue(node)) {
                if (nanosTimeout <= 0L) {
                    timedout = transferAfterCancelledWait(node);
                    break;
                }
                if (nanosTimeout >= spinForTimeoutThreshold)
                    LockSupport.parkNanos(this, nanosTimeout);
                if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
                    break;
                nanosTimeout = deadline - System.nanoTime();
            }
            if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
                interruptMode = REINTERRUPT;
            if (node.nextWaiter != null)
                unlinkCancelledWaiters();
            if (interruptMode != 0)
                reportInterruptAfterWait(interruptMode);
            return !timedout;
        }

        //  support for instrumentation

        /**
         * Returns true if this condition was created by the given
         * synchronization object.
         *
         * @return {@code true} if owned
         */
        final boolean isOwnedBy(AbstractQueuedSynchronizer sync) {
            return sync == AbstractQueuedSynchronizer.this;
        }

        /**
         * Queries whether any threads are waiting on this condition.
         * Implements {@link AbstractQueuedSynchronizer#hasWaiters(ConditionObject)}.
         *
         * @return {@code true} if there are any waiting threads
         * @throws IllegalMonitorStateException if {@link #isHeldExclusively}
         *         returns {@code false}
         */
        protected final boolean hasWaiters() {
            if (!isHeldExclusively())
                throw new IllegalMonitorStateException();
            for (Node w = firstWaiter; w != null; w = w.nextWaiter) {
                if (w.waitStatus == Node.CONDITION)
                    return true;
            }
            return false;
        }

        /**
         * Returns an estimate of the number of threads waiting on
         * this condition.
         * Implements {@link AbstractQueuedSynchronizer#getWaitQueueLength(ConditionObject)}.
         *
         * @return the estimated number of waiting threads
         * @throws IllegalMonitorStateException if {@link #isHeldExclusively}
         *         returns {@code false}
         */
        protected final int getWaitQueueLength() {
            if (!isHeldExclusively())
                throw new IllegalMonitorStateException();
            int n = 0;
            for (Node w = firstWaiter; w != null; w = w.nextWaiter) {
                if (w.waitStatus == Node.CONDITION)
                    ++n;
            }
            return n;
        }

        /**
         * Returns a collection containing those threads that may be
         * waiting on this Condition.
         * Implements {@link AbstractQueuedSynchronizer#getWaitingThreads(ConditionObject)}.
         *
         * @return the collection of threads
         * @throws IllegalMonitorStateException if {@link #isHeldExclusively}
         *         returns {@code false}
         */
        protected final Collection<Thread> getWaitingThreads() {
            if (!isHeldExclusively())
                throw new IllegalMonitorStateException();
            ArrayList<Thread> list = new ArrayList<Thread>();
            for (Node w = firstWaiter; w != null; w = w.nextWaiter) {
                if (w.waitStatus == Node.CONDITION) {
                    Thread t = w.thread;
                    if (t != null)
                        list.add(t);
                }
            }
            return list;
        }
    }
```

#### 内部方法（private）

#### addConditionWaiter

```java
        /**
         * Adds a new waiter to wait queue.
         * @return its new wait node
         */
      	// 添加新的waiter到wait队列
        private Node addConditionWaiter() {
          	// 保存尾结点
            Node t = lastWaiter;
            // If lastWaiter is cancelled, clean out.
            if (t != null && t.waitStatus != Node.CONDITION) {// 尾结点不为空，并且尾结点的状态不为CONDITION
                unlinkCancelledWaiters(); // 清除状态不为CONDITION(为CANCELLED)的结点
                t = lastWaiter;
            }
            Node node = new Node(Thread.currentThread(), Node.CONDITION); // 新建一个结点
            if (t == null) // 尾结点为空
                firstWaiter = node; // 设置condition队列的头结点
            else // 尾结点不为空，说明队列不为空
                t.nextWaiter = node; // 设置为节点的nextWaiter域为node结点
            lastWaiter = node; // 更新node结点为condition队列的尾结点
            return node;
        }
```

#### doSignal

```java
        /**
         * Removes and transfers nodes until hit non-cancelled one or
         * null. Split out from signal in part to encourage compilers
         * to inline the case of no waiters.
         * @param first (non-null) the first node on condition queue
         */
      	// 把Condition队列第一个不为空的结点，转移到Sync结点
        private void doSignal(Node first) {
            do { // 循环
                if ( (firstWaiter = first.nextWaiter) == null) // 该节点的nextWaiter为空
                    lastWaiter = null; // 设置尾结点为空 
                first.nextWaiter = null; // 设置first结点的nextWaiter域
            } while (!transferForSignal(first) && // 将结点从condition队列转移到sync队列失败
                     (first = firstWaiter) != null); // 并且condition队列中的头结点不为空，一直循环
        }
```

#### doSignalAll

```java
        /**
         * Removes and transfers all nodes.
         * @param first (non-null) the first node on condition queue
         */
        // 将CONDITION队列中的全部结点移除并加入到同步队列中竞争同步状态
        private void doSignalAll(Node first) {
            // condition队列的头结点尾结点都设置为空
            lastWaiter = firstWaiter = null;
            do {// 循环
                Node next = first.nextWaiter; // 获取first结点的nextWaiter域结点
                first.nextWaiter = null; // 设置first结点的nextWaiter域为空(切断第一个结点)
                transferForSignal(first); // 将first结点从condition队列转移到sync队列
                first = next; // 重新设置first
            } while (first != null);
        }
```

#### unlinkCancelledWaiters

```java
        /**
         * Unlinks cancelled waiter nodes from condition queue.
         * Called only while holding lock. This is called when
         * cancellation occurred during condition wait, and upon
         * insertion of a new waiter when lastWaiter is seen to have
         * been cancelled. This method is needed to avoid garbage
         * retention in the absence of signals. So even though it may
         * require a full traversal, it comes into play only when
         * timeouts or cancellations occur in the absence of
         * signals. It traverses all nodes rather than stopping at a
         * particular target to unlink all pointers to garbage nodes
         * without requiring many re-traversals during cancellation
         * storms.
         */
      	// 清除condition队列中，状态为为CANCELLED的结点
        private void unlinkCancelledWaiters() {
            Node t = firstWaiter; // 保存condition队列头结点
            Node trail = null; 
            while (t != null) { // t不为空
                Node next = t.nextWaiter; // 下一个结点
                if (t.waitStatus != Node.CONDITION) { // t结点的状态不为CONDTION状态（为CANCELLED）
                    t.nextWaiter = null; // 设置t节点的nextWaiter为空
                    if (trail == null)
                        firstWaiter = next;
                    else
                        trail.nextWaiter = next;
                    if (next == null)
                        lastWaiter = trail;
                }
                else // t结点的状态为CONDTION状态
                    trail = t; // 设置trail结点
                t = next; // 设置t结点为下一个结点
            }
        }
```

#### 公共方法 （public）

#### signal

```java
        /**
         * Moves the longest-waiting thread, if one exists, from the
         * wait queue for this condition to the wait queue for the
         * owning lock.
         *
         * @throws IllegalMonitorStateException if {@link #isHeldExclusively}
         *         returns {@code false}
         */
      	// 将等待时间最长的线程（如果存在）从此条件的等待队列移动到拥有锁的等待队列。
        // 唤醒一个等待线程。如果所有的线程都在等待此条件，则选择其中的一个唤醒。在从 await 返回之前，该线程必须重新获取锁。
        public final void signal() {
            if (!isHeldExclusively()) // 不被当前线程独占（非独占模式），抛出异常
                throw new IllegalMonitorStateException();
            Node first = firstWaiter; // 获取头结点
            if (first != null) 
                doSignal(first);  // 把Condition队列第一个不为空的结点，转移到Sync结点
        }
```

#### signalAll

```java
        /**
         * Moves all threads from the wait queue for this condition to
         * the wait queue for the owning lock.
         *
         * @throws IllegalMonitorStateException if {@link #isHeldExclusively}
         *         returns {@code false}
         */
      	// 将所有结点从等待队列移动到拥有锁的等待队列。
      	// 唤醒所有等待线程。如果所有的线程都在等待此条件，则唤醒所有线程。在从 await 返回之前，每个线程都必须重新获取锁。
        public final void signalAll() {
            if (!isHeldExclusively())  // 不被当前线程独占（非独占模式），抛出异常
                throw new IllegalMonitorStateException();
            Node first = firstWaiter;
            if (first != null)
                doSignalAll(first); // 将CONDITION队列中的全部结点移除并加入到同步队列中竞争同步状态
        }
```

#### await

使当前线程等待，直到它收到信号或被中断

```java
        /**
         * Implements interruptible condition wait.
         * <ol>
         * <li> If current thread is interrupted, throw InterruptedException.
         * <li> Save lock state returned by {@link #getState}.
         * <li> Invoke {@link #release} with saved state as argument,
         *      throwing IllegalMonitorStateException if it fails.
         * <li> Block until signalled or interrupted.
         * <li> Reacquire by invoking specialized version of
         *      {@link #acquire} with saved state as argument.
         * <li> If interrupted while blocked in step 4, throw InterruptedException.
         * </ol>
         */
        public final void await() throws InterruptedException {
            if (Thread.interrupted())  // 当前线程被中断，抛出异常
                throw new InterruptedException();
            Node node = addConditionWaiter(); // 在wait队列上添加一个结点
            int savedState = fullyRelease(node); // 获取释放的状态（激活后置结点的状态）
            int interruptMode = 0;
            while (!isOnSyncQueue(node)) { // 判断节点是否在CLH队列中（不在队列中），说明该线程还不具备竞争锁的资格
                LockSupport.park(this);   // 阻塞当前线程
                if ((interruptMode = checkInterruptWhileWaiting(node)) != 0) // 如果线程中断，退出
                    break;
            }
                      // 上面的循环退出有两种情况：
            // 1. isOnSyncQueue(node) 为true，即当前的node已经转移到CLH队列了
            // 2. checkInterruptWhileWaiting != 0, 表示线程中断
          
            if (acquireQueued(node, savedState) && interruptMode != THROW_IE) // 退出循环，被唤醒之后，进入阻塞队列，等待获取锁 acquireQueued
                interruptMode = REINTERRUPT;
            if (node.nextWaiter != null) // clean up if cancelled
                unlinkCancelledWaiters(); // 线程中断，清除condition队列中，状态为为CANCELLED的结点
            if (interruptMode != 0)
                reportInterruptAfterWait(interruptMode);
        }
```

#### awaitNanos

使当前线程等待，直到它被发出信号或被中断，或者指定的等待时间过去

 所有 awaitXX 方法其实就是 	

1. 将当前的线程封装成 Node 加入到 Condition 里面;
2. 丢到当前线程所拥有的独占锁
3. 等待 其他获取 独占锁的线程的唤醒, 唤醒从 Condition Queue 到 Sync Queue 里面, 进而获取独占锁 
4. 最后获取 lock 之后, 在根据线程唤醒的方式(signal/interrupt) 进行处理 *  4. 最后还是需要调用 lock./unlock 进行释放锁

```java
        /**
         * Implements timed condition wait.
         * <ol>
         * <li> If current thread is interrupted, throw InterruptedException.
         * <li> Save lock state returned by {@link #getState}.
         * <li> Invoke {@link #release} with saved state as argument,
         *      throwing IllegalMonitorStateException if it fails.
         * <li> Block until signalled, interrupted, or timed out.
         * <li> Reacquire by invoking specialized version of
         *      {@link #acquire} with saved state as argument.
         * <li> If interrupted while blocked in step 4, throw InterruptedException.
         * </ol>
         */
        public final long awaitNanos(long nanosTimeout)
                throws InterruptedException {
            if (Thread.interrupted())
                throw new InterruptedException();
            Node node = addConditionWaiter(); // 将结点添加到Condition等待队列
            int savedState = fullyRelease(node); // 获取释放的状态（激活后置结点的状态）
            final long deadline = System.nanoTime() + nanosTimeout; // 等待截止时间
            int interruptMode = 0; 
            while (!isOnSyncQueue(node)) { // 判断节点是否在CLH队列中（不在队列中）
                if (nanosTimeout <= 0L) { // 等待时间到了
                    transferAfterCancelledWait(node);//将Node 从 Condition 转移到 Sync Queue 里面
                    break;
                }
           // 当剩余时间 < spinForTimeoutThreshold, 其实函数 spin 比用 LockSupport.parkNanos 更高效
                if (nanosTimeout >= spinForTimeoutThreshold) 
                    LockSupport.parkNanos(this, nanosTimeout); // 进行线程的 block
              // 判断此次线程的唤醒是否因为线程被中断, 若是被中断, 则会在checkInterruptWhileWaiting的transferAfterCancelledWait 进行节点的转移; 返回值 interruptMode != 0
                if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
                    break;
                nanosTimeout = deadline - System.nanoTime(); // 更新一下还需要等待多久
            }
          // 调用独占锁的获取, 返回值表明在获取的过程中有没有被中断过
            if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
                interruptMode = REINTERRUPT;
          // 通过 "node.nextWaiter != null" 判断 线程的唤醒是中断还是 signal, 因为通过中断唤醒的话, 此刻代表线程的 Node 在 Condition Queue 与 Sync Queue 里面都会存在
            if (node.nextWaiter != null)
                unlinkCancelledWaiters(); // 进行 cancelled 节点的清除
            if (interruptMode != 0) // 代表通过中断的方式唤醒线程
              // 根据 interruptMode 的类型决定是抛出异常, 还是自己再中断一下
                reportInterruptAfterWait(interruptMode); 
            return deadline - System.nanoTime(); // 这个返回值代表是 通过 signal;还是超时
        } 
```



#### awaitUninterruptibly

使当前线程等待，直到它收到信号（不会响应中断）

```java
        /**
         * Implements uninterruptible condition wait.
         * <ol>
         * <li> Save lock state returned by {@link #getState}.
         * <li> Invoke {@link #release} with saved state as argument,
         *      throwing IllegalMonitorStateException if it fails.
         * <li> Block until signalled.
         * <li> Reacquire by invoking specialized version of
         *      {@link #acquire} with saved state as argument.
         * </ol>
         */
				// 等待，当前线程在接到信号之前一直处于等待状态，不响应中断
        public final void awaitUninterruptibly() {
            Node node = addConditionWaiter(); // 将结点添加到Condition等待队列
            int savedState = fullyRelease(node); // 获取释放的状态（激活后置结点的状态）
            boolean interrupted = false; 
            while (!isOnSyncQueue(node)) { // 判断节点是否在CLH队列中（不在队列中）
                LockSupport.park(this); // 阻塞当前线程
                if (Thread.interrupted())
                    interrupted = true;
            }
            if (acquireQueued(node, savedState) || interrupted) // 以独占不间断模式获取已在队列中的阻塞线程
                selfInterrupt();
        }
```



## 引用

- [Java并发包源码学习系列：AbstractQueuedSynchronizer](https://www.cnblogs.com/summerday152/p/14238284.html)

- [AQS原理学习笔记](https://juejin.cn/post/6844903782636060679#heading-4)

- [一行一行源码分析清楚AbstractQueuedSynchronizer](https://javadoop.com/post/AbstractQueuedSynchronizer)

- [Java并发包源码学习系列：详解Condition条件队列、signal和await](

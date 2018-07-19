### AbstractQueuedSynchronizer
* 提供了一个非阻塞的FIFO队列（在操作插入或删除元素时，并发条件下不会阻塞，而是通过自旋锁和CAS保证操作的原子性），可以用于构造锁和基本同步器的基础框架，是JUC中几乎所有锁的基础。
* FIFO队列底层中通过`head`和`tail`维护一个以内部类`Node`为元素的类LCH队列，
* 除此之外使用内部类ConfitionObject实现一个等待条件的单向队列。
* 在线程获取到锁之后会检查条件是否满足，若不满足则先进入到条件等待队列，满足后从该队列入总的锁等待队列。
* 对上层实现提供了几种必要需求
    1. 独占锁和共享锁两种机制
    2. 线程阻塞后，支持中断。
    3. 支持阻塞并超时后中断的机制

#### 成员变量
```java
         private transient volatile Node head;     // 底层FIFO队列的头节点
         private transient volatile Node tail;     // 尾节点
         private volatile int state;               // 当前同步器状态，通过CAS和volatile保证其原子性和可见性
         // RetrantLock可以将这个state用于存储当前线程的重进入次数，Semaphore可以用这个state存储许可数，CountDownLatch则可以存储需要被countDown的次数，而Future则可以存储当前任务的执行状态(RUNING,RAN,CANCELL)
```

#### **Node** 底层的等待队列的元素
```java
    static final class Node {
            // 共享模式
            static final Node SHARED = new Node();
            // 独占模式
            static final Node EXCLUSIVE = null;        
            // 结点状态
            // CANCELLED  表示当前的线程被取消
            // SIGNAL     表示当前节点的后继节点包含的线程需要运行
            // CONDITION  表示当前节点在等待condition
            // PROPAGATE  表示当前场景下后续的acquireShared能够得以执行
            // 值为0，表示当前节点在sync队列中，等待着获取锁
            static final int CANCELLED =  1;
            static final int SIGNAL    = -1;
            static final int CONDITION = -2;
            static final int PROPAGATE = -3;        
            // 结点状态
            volatile int waitStatus;        
            // 前驱结点
            volatile Node prev;    
            // 后继结点
            volatile Node next;        
            // 结点所对应的线程
            volatile Thread thread;        
            // 下一个等待者
            Node nextWaiter;
            
            // 结点是否在共享模式下等待
            final boolean isShared() {
                return nextWaiter == SHARED;
            }
            
            // 获取前驱结点，若前驱结点为空，抛出异常
            final Node predecessor() throws NullPointerException {
                // 保存前驱结点
                Node p = prev; 
                if (p == null) // 前驱结点为空，抛出异常
                    throw new NullPointerException();
                else // 前驱结点不为空，返回
                    return p;
            }
            
            // 无参构造函数
            Node() {    // Used to establish initial head or SHARED marker
            }
            
            // 构造函数
             Node(Thread thread, Node mode) {    // Used by addWaiter
                this.nextWaiter = mode;
                this.thread = thread;
            }
            
            // 构造函数
            Node(Thread thread, int waitStatus) { // Used by Condition
                this.waitStatus = waitStatus;
                this.thread = thread;
            }
        }

```

#### **ConditionObject** 等待队列,实现了Condition接口
- Condition接口的方法实现
```java
        /**
          * 该方法会使当前线程释放锁 并进入Condition对应的等待队列，该线程转为等待状态
          */
      public final void await() throws InterruptedException {
                // 抛出中断异常
               if (Thread.interrupted())
                   throw new InterruptedException();
               Node node = addConditionWaiter();                                       // 当前节点入队列，并返回
               int savedState = fullyRelease(node);                                    // 释放当前线程占用的资源，并返回调用时的状态
               int interruptMode = 0;
               while (!isOnSyncQueue(node)) {
                   LockSupport.park(this);
                   if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
                       break;
               }
               if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
                   interruptMode = REINTERRUPT;
               if (node.nextWaiter != null) // clean up if cancelled
                   unlinkCancelledWaiters();
               if (interruptMode != 0)
                   reportInterruptAfterWait(interruptMode);
        }
         /**
           *  添加到等待队列 
           */
        private Node addConditionWaiter() {
                    // 取得最后一个等待线程
                    Node t = lastWaiter;
                    // If lastWaiter is cancelled, clean out.
                    if (t != null && t.waitStatus != Node.CONDITION) {
                        unlinkCancelledWaiters();                                 // 剔除取消状态的节点
                        t = lastWaiter;                                           // t指向最后一个节点
                    }
                    Node node = new Node(Thread.currentThread(), Node.CONDITION); // 构建新的节点
                    if (t == null)                                                // t为空表示队列里面没有元素
                        firstWaiter = node;                 
                    else
                        t.nextWaiter = node;
                    lastWaiter = node;
                    return node;
                }
                /**
                  *  循环遍历等代列表 剔除不处于CONDITION状态的节点
                  */
         private void unlinkCancelledWaiters() {
                    // 获得第一个等待节点
                    Node t = firstWaiter;
                    Node trail = null;                          // 追踪节点
                    while (t != null) { 
                        Node next = t.nextWaiter;               // 获取下一个节点
                        if (t.waitStatus != Node.CONDITION) {   // 状态不是-3
                            t.nextWaiter = null;                // 头结点的下一个节点置为空
                            if (trail == null)                  // 当追踪节点为空,后继节点变为头结点
                                firstWaiter = next;
                            else
                                trail.nextWaiter = next;     
                            if (next == null)
                                lastWaiter = trail;
                        }
                        else
                            trail = t;
                        t = next;
                    }
                }
            /**
              *     以当前同步器的状态调用release方法，
              *     该方法主要作用就是为调用release准备param，以及做失败之后的处理
              */
         final int fullyRelease(Node node) {
                boolean failed = true;
                try {
                    int savedState = getState();                
                    if (release(savedState)) {                     // 主要代码，调用release
                        failed = false;
                        return savedState;
                    } else {                                        // release失败则抛出错误
                        throw new IllegalMonitorStateException();
                    }
                } finally {
                    if (failed)                                     // 失败之后将当前节点设为取消状态
                        node.waitStatus = Node.CANCELLED;
                }
            }

```

#### 成员方法 
- acquire  该方法用于以独占形式获取锁
```java
    /**
      *  尝试以独占形式获取锁
      *  在ReentrantLock中是锁已经被占用的情况下调用
      */
    public final void acquire(int arg) {
        if (!tryAcquire(arg) &&
            acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
            selfInterrupt();
    }
    /**
     * 受保护的类,tryAcquire顾名思义为尝试获取资源， true表示获取成功
     * 在AbstractQueuedSunchronizer中仅仅是抛出错误，具体的获取逻辑在子类中实现
     */
    protected boolean tryAcquire(int arg) {
            throw new UnsupportedOperationException();
    }
    /**
      * 该方法用于以指定模式添加当前线程到等待队列
      * return: 返回实际入队节点
      */
     private Node addWaiter(Node mode) {
            // 创建node私立
            Node node = new Node(Thread.currentThread(), mode);
            // Try the fast path of enq; backup to full enq on failure
            // 获得尾节点
            Node pred = tail;
            // 如果尾节点不为空
            if (pred != null) {
                // 当前节点的上一个节点指向尾节点
                node.prev = pred;
                // 比较pred是否是尾节点,是就讲node置为尾节点
                if (compareAndSetTail(pred, node)) {
                    // pred下一个节点指向当前节点
                    pred.next = node;
                    return node;
                }
            }
            // 在尾节点为空时
            enq(node);
            return node;
        }
    /**
      *  将node入队，必要时创建队列
      *  return： 返回入队节点的前驱节点
      */
     private Node enq(final Node node) {
            // 无限循环确保成功入队
            for (;;) {
                // 新建node指向尾节点
                Node t = tail;
                // 尾节点为空时，必须创建!!
                if (t == null) { // Must initialize
                    // 若头结点为空则将head置为当前node
                    if (compareAndSetHead(new Node()))
                        // 首尾一致
                        tail = head;
                } else {    // 头节点不为空时
                    // 将当前节点的上一个节点置为尾节点
                    node.prev = t;
                    // cas比较并修改尾节点为当前节点
                    if (compareAndSetTail(t, node)) {
                        // 尾节点的下一个节点为当前节点
                        t.next = node;
                        return t;
                    }
                }
            }
        }
    /**
      *  线程在等待队列中也是一直在尝试获取锁，直到获取到锁
      *  如果其中被中断则返回true
      */
    final boolean acquireQueued(final Node node, int arg) {
            // 失败标识
            boolean failed = true;
            try {
                // 是否中断标志
                boolean interrupted = false;
                for (;;) {              // 自旋  ！！！此处极其重要，线程在等待队列里面也是一直在尝试获取锁的。
                    // 获取前驱节点
                    final Node p = node.predecessor();
                    // 如果p为头结点表示当前节点可以尝试获取锁
                    if (p == head && tryAcquire(arg)) {
                        // 将该节点置为头结点 表示成功获取到锁
                        setHead(node);  
                        // 头结点的next为空 帮助GC回收
                        p.next = null; // help GC
                        failed = false;
                        return interrupted;
                    }
                    if (shouldParkAfterFailedAcquire(p, node) &&
                        parkAndCheckInterrupt())
                        interrupted = true;
                }
            } finally {     // 当前驱节点为空时会抛出NPE
                if (failed)
                    cancelAcquire(node);
            }
        }
         /**
           *    检查并更新获取失败后的节点状态,将前驱节点置为Signal,此外此方法会清除取消状态的节点
           */
     private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
            int ws = pred.waitStatus;   // 获取前驱节点的状态
            // 前驱节点的状态为Signal
            if (ws == Node.SIGNAL)  
                return true;
            // ws > 0表示前驱节点的状态为CANCELLED
            if (ws > 0) {
                // 循环遍历找到不是取消状态的节点,取消表示前驱不在尝试获取锁
                do {
                    node.prev = pred = pred.prev;
                } while (pred.waitStatus > 0);
                // 找到前一个不在取消的锁并排在他的后面
                pred.next = node;
            } else {
                // 如果前驱状态不是取消，则改变前驱状态为Signal，即表示本节点需要锁
                compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
            }
            return false;
     }
      /**
        *  使线程进入等待状态
        */
     private final boolean parkAndCheckInterrupt() {
            LockSupport.park(this); //调用park()使线程进入waiting状态
            return Thread.interrupted();
        }
```

> 整个acquire的流程
> 1. 调用tryAcquire方法尝试获取锁
> 2. 获取锁失败之后,先调用addWaiter方法以独占方式将锁加入到等待队列
> 3. acquireQueued中，线程无限循环尝试获取锁资源，可响应中断。

- release 释放独占模式下的锁
```java
    public final boolean release(int arg) {
        // 尝试释放锁 失败进入if代码
        if (tryRelease(arg)) {
            // 获取头节点
            Node h = head;      
            // 如果头节点不为空 且想头节点的状态不是等待获取锁
            if (h != null && h.waitStatus != 0)
                // 唤醒队列中下一个等待的线程
                unparkSuccessor(h);
            return true;
        }
        return false;
    }
    
    /**
     * 尝试释放共享资源的锁，具体的逻辑实现由子类完成 
     * return:true为已经释放
     */
    protected boolean tryRelease(int arg) {
        throw new UnsupportedOperationException();
    }
       /**
        *  唤醒等待队列重下一个线程  
        */
     private void unparkSuccessor(Node node) {
            ws = node.waitStatus;
            // 判断并置0当前线程状态(取消状态不论)
            if (ws < 0)
                compareAndSetWaitStatus(node, ws, 0);
            // 获取后继节点
            Node s = node.next;
            // 后继节点为空 || 后继节点的状态为取消 
            if (s == null || s.waitStatus > 0) {
                s = null;
                // 从队尾开始向前找第一个有效节点
                for (Node t = tail; t != null && t != node; t = t.prev)
                    // 判断是否是有效节点
                    if (t.waitStatus <= 0)    
                        s = t;
            }
            if (s != null)
                LockSupport.unpark(s.thread);
        }
    
```

> release的流程
> 1. tryRelease尝试释放资源,若释放成功返回true
> 2. 释放成功之后调用unparkSuccessor唤醒队列中下一个等待线程

- acquireShared   该方法以共享的形式获取锁
```java
    /**
      *  以共享的形式获取锁
      */
    public final void acquireShared(int arg) {
        // 尝试获取锁
        if (tryAcquireShared(arg) < 0)
            doAcquireShared(arg);
    }
    /**
      *  尝试获取锁的方法,具体实现在子类
      *  因为在acquireShared中的判断,所以对成功或者失败的返回值有限制
      *  成功负数代表获取失败，0表示获取成功,但没有剩余资源,正数代表获取成功还有资源
      */
    protected int tryAcquireShared(int arg) {
            throw new UnsupportedOperationException();
    }
    /**
      *  共享模式下使当前线程进入等待队列(进入队尾) 
      */
    private void doAcquireShared(int arg) {
            // 创建共享模式的node实例
            final Node node = addWaiter(Node.SHARED);
            // 是否成功标识
            boolean failed = true;
            try {
                // 是否被中断标识
                boolean interrupted = false;
                for (;;) {      // 无限循环,确保进入队列
                    // 获得前驱节点,为空抛NPE
                    final Node p = node.predecessor();
                    if (p == head) {    // 如果node的前驱节点是head,node处于第二则有机会获得资源
                        // 尝试获取资源
                        int r = tryAcquireShared(arg);
                        if (r >= 0) {   // 正数代表资源还有剩余
                            setHeadAndPropagate(node, r);
                            p.next = null; // help GC
                            if (interrupted)
                                selfInterrupt();
                            failed = false;
                            return;
                        }
                    }
                    if (shouldParkAfterFailedAcquire(p, node) &&
                        parkAndCheckInterrupt())
                        interrupted = true;
                }
            } finally {                         // 退出循环时没有获取到锁，则置为取消状态
                if (failed)
                    cancelAcquire(node);
            }
        }
        /**
          *    设置头节点并唤醒可以共享的线程
          */
         private void setHeadAndPropagate(Node node, int propagate) {
                Node h = head; // Record old head for check below
                setHead(node);   
                // 操作太骚 我看不懂
                if (propagate > 0 || h == null || h.waitStatus < 0 ||
                    (h = head) == null || h.waitStatus < 0) {
                    Node s = node.next;
                    if (s == null || s.isShared())
                        doReleaseShared();
                }
            }
```

- 其实共享形式和独占形式在代码上大同小异,在共享形式的获取锁完成之后会尝试唤醒别的线程，主要还是四个线程状态。
 
 * 线程状态
    
| 状态 | 含义 | 数值 |
| ----  | ----   | ----  | 
| CANCELLED  | 等待队列中的线程被中断或者超时,会变为取消状态 | 1 |
| SIGNAL     | 表示该节点的后继节点等待唤醒，在完成该节点后会唤醒后继节点|-1|
| CONDITION  | 该节点位于条件等待队列,当其他线程调用了`condition.signal()`方法后会被唤醒进入同步队列 | -2|
| PROPAGATE  | 共享模式中，该状态的节点处于Runnable状态            | -3 |
| 初始状态    | 初始化状态                                          | 0 |

** 表格第二行的短横少加一个竟然就不能显示为表格 **





- - - 
#### AbstractOwnableSynchonizer

```java 
 /**
   * A synchronizer that may be exclusively owned by a thread.  a
   * This class provides a basis for creating locks and related synchronizers 
   * that may entail a notion of ownership. 
   */
```
* 该类主要定义了线程独占的方式拥有的同步器，提供了创建锁和相关同步器的基础，并且可能会涉及到所有权的概念。

```java
    public abstract class AbstractOwnableSynchronizer
            implements java.io.Serializable {
        private transient Thread exclusiveOwnerThread;      // 独占线程
        // 设置当前拥有独占访问权的线程
        protected final void setExclusiveOwnerThread(Thread thread) {
                exclusiveOwnerThread = thread;
        }
        // 获取当前拥有独占访问权的线程
        protected final Thread getExclusiveOwnerThread() {
                return exclusiveOwnerThread;
        }
    }
```
* 由源码可见在AbstractOwnableSynchronizer中并不提供对该独占线程的管理


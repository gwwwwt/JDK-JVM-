## 引用相关

参考：

1. [http://imushan.com/2018/08/19/java/language/JDK%E6%BA%90%E7%A0%81%E9%98%85%E8%AF%BB-Reference/](http://imushan.com/2018/08/19/java/language/JDK源码阅读-Reference/)

2. https://zhuanlan.zhihu.com/p/77357666





### 1. Java 中有四种引用类

> + 强引用(Strong Reference)：普通的的引用类型，new一个对象默认得到的引用就是强引用，只要对象存在强引用，就不会被GC。
>
> + 软引用(Soft Reference)：相对较弱的引用，垃圾回收器会在内存不足时回收弱引用指向的对象。JVM会在抛出OOME前清理所有弱引用指向的对象，如果清理完还是内存不足，才会抛出OOME。所以软引用一般用于实现内存敏感缓存。
>
> + 弱引用(Weak Reference)：更弱的引用类型，垃圾回收器在GC时会回收此对象，也可以用于实现缓存，比如JDK提供的WeakHashMap。
>
> + 虚引用(Phantom Reference)：一种特殊的引用类型，不能通过虚引用获取到关联对象，只是用于获取对象被回收的通知。
>
>   注: 虚引用可以用在管理堆外内存，见DirectByteBuffer中的Cleaner



四种引用决定了五种类型的可达性

> 1. 强可达(Strongly Reachable)：如果线程能通过强引用访问到对象，那么这个对象就是强可达的。
> 2. 软可达(Soft Reachable)：如果一个对象不是强可达的，但是可以通过软引用访问到，那么这个对象就是软可达的
> 3. 弱可达(Weak Reachable)：如果一个对象不是强可达或者软可达的，但是可以通过弱引用访问到，那么这个对象就是弱可达的。
> 4. 虚可达(Phantom Reachable)：如果一个对象不是强可达，软可达或者弱可达，并且这个对象已经finalize过了，并且有虚引用指向该对象，那么这个对象就是虚可达的。
> 5. 不可达(Unreachable)：如果对象不能通过上述的几种方式访问到，则对象是不可达的，可以被回收。

如例：

![类型可达性](http://imushan.com/img/image-20180820224722593.png)

> - A依然只有一个强引用，所以A是强可达
> - B存在两个引用，强引用和软引用，但是B可以通过强引用访问到，所以B是强可达
> - C只能通过弱引用访问到，所以是弱可达
> - D存在弱引用和虚引用，所以是弱可达
> - E虽然存在F的强引用，但是GC Root无法访问到它，所以它依然是不可达。



### 2. Reference总体结构

![Reference类层次结构](http://imushan.com/img/image-20180819111336990.png)

>- `SoftReference`：软引用，堆内存不足时，垃圾回收器会回收对应引用
>- `WeakReference`：弱引用，每次垃圾回收都会回收其引用
>- `PhantomReference`：虚引用，对引用无影响，只用于获取对象被回收的通知
>- `FinalReference`：Java用于实现finalization的一个内部类
>- 因为默认的引用就是强引用，所以没有强引用的Reference实现类。



### 3. Reference源码



```java
public abstract class Reference<T> {
    // referent只是一个普通的强引用，只是JVM垃圾回收器中通过硬编码识别SoftReference等类型，从而会被GC特殊对待
	  private T referent; 
    // reference被回收后，当前Reference实例会被添加到这个队列中
    volatile ReferenceQueue<? super T> queue;
  
    // 全局唯一的pending-Reference列表
    private static Reference<Object> pending = null;
    
    // Reference为Active：由垃圾回收器管理的已发现的引用列表(这个不在本文讨论访问内)
    // Reference为Pending：在pending列表中的下一个元素，如果没有为null
    // 其他状态：NULL
    transient private Reference<T> discovered;  /* used by VM */
  
    /* When active:   NULL
     *     pending:   this
     *    Enqueued:   指向ReferenceQueue中的下一个元素，如果没有，指向this
     *    Inactive:   this
     */
    Reference next;
    
    // 只传入reference的构造函数，意味着用户只需要特殊的引用类型，不关心对象何时被GC
    Reference(T referent) {
        this(referent, null);
    }
	
    // 传入referent和ReferenceQueue的构造函数，reference被回收后，会添加到queue中
    Reference(T referent, ReferenceQueue<? super T> queue) {
        this.referent = referent;
        this.queue = (queue == null) ? ReferenceQueue.NULL : queue;
    }
    
    // ...
}
```



> Reference及其子类的两大功能：
>
> 1. 实现特定的引用类型
>
>    > 如上面的注释，JVM通过硬编码识别具体的Reference类型，从而采取不同的GC策略
>
> 2. 用户可以对象被回收后得到通知
>
>    >实现这个功能有两种思路
>    >
>    >- 在Reference定义一个回调方法，由用户通过构造方法或setter方法设置。这样当`java.lang.ref.Reference#referent`被回收时，JVM调用该回调，这种思路比较符合一般的通知模型，但是对于引用与垃圾回收这种底层场景来说，会导致实现复杂，性能不高的问题，比如需要考虑在什么线程中执行这个回调，回调执行阻塞怎么办等等。所以没有采用这个方案;
>    >- 把引用对象被回收的`Reference`添加到一个队列中，用户后续自己去从队列中获取并使用。所以`Reference`有一个`queue`成员变量，用于存储引用对象被回收`Reference`实例



### 4. Reference实例如何添加到ReferenceQueue

> + Reference可能的状态
>
>   > 1. `Active`：新创建的实例的状态，由垃圾回收器进行处理，如果实例的可达性处于合适的状态，垃圾回收器会切换实例的状态为`Pending`或者`Inactive`。如果Reference注册了ReferenceQueue，则会切换为`Pending`，并且Reference会加入`pending-Reference`链表中，如果没有注册ReferenceQueue，会切换为`Inactive`。
>   > 2. `Pending`：在`pending-Reference`链表中的Reference的状态，这些Reference等待被加入ReferenceQueue中。
>   > 3. `Enqueued`：在ReferenceQueue队列中的Reference的状态，如果Reference从队列中移除，会进入`Inactive`状态
>   > 4. `Inactive`：Reference的最终状态
>
>   ![Reference状态转换](http://imushan.com/img/image-20180820230137796.png)



> + pending-Reference
>
>   除了上文提到的`ReferenceQueue`，这里出现了一个新的数据结构：`pending-Reference`。这个链表是用来干什么的呢？
>
>   上文提到了，`reference`引用的对象被回收后，该`Reference`实例会被添加到`ReferenceQueue`中，但是这个不是垃圾回收器来做的，这个操作还是有一定逻辑的，如果垃圾回收器还需要执行这个操作，会降低其效率。从另外一方面想，`Reference`实例会被添加到`ReferenceQueue`中的实效性要求不高，所以也没必要在回收时立马加入`ReferenceQueue`。
>
>   所以垃圾回收器做的是一个更轻量级的操作：把`Reference`添加到`pending-Reference`链表中。`Reference`对象中有一个pending成员变量，是静态变量，它就是这个`pending-Reference`链表的头结点。要组成链表，还需要一个指针，指向下一个节点，这个对应的是`java.lang.ref.Reference#discovered`这个成员变量。



> + ReferenceHandler线程
>
>   通过上文的讨论，我们知道一个Reference实例化后状态为Active，其引用的对象被回收后，垃圾回收器将其加入到`pending-Reference`链表，等待加入ReferenceQueue。这个过程是如何实现的呢？
>
>   这个过程不能对垃圾回收器产生影响，所以不能在垃圾回收线程中执行，也就需要一个独立的线程来负责。这个线程就是`ReferenceHandler`，它定义在`Reference`类中：
>
>   ```java
>   // 用于控制垃圾回收器操作与Pending状态的Reference入队操作不冲突执行的全局锁
>   // 垃圾回收器开始一轮垃圾回收前要获取此锁
>   // 所以所有占用这个锁的代码必须尽快完成，不能生成新对象，也不能调用用户代码
>   static private class Lock { };
>   private static Lock lock = new Lock();
>   
>   private static class ReferenceHandler extends Thread {
>   
>       ReferenceHandler(ThreadGroup g, String name) {
>           super(g, name);
>       }
>   
>       public void run() {
>           // 这个线程一直执行
>           for (;;) {
>               Reference<Object> r;
>               // 获取锁，避免与垃圾回收器同时操作
>               synchronized (lock) {
>                   // 判断pending-Reference链表是否有数据
>                   if (pending != null) {
>                       // 如果有Pending Reference，从列表中取出
>                       r = pending;
>                       pending = r.discovered;
>                       r.discovered = null;
>                   } else {
>                       // 如果没有Pending Reference，调用wait等待
>                       // 
>                       // wait等待锁，是可能抛出OOME的，
>                       // 因为可能发生InterruptedException异常，然后就需要实例化这个异常对象，
>                       // 如果此时内存不足，就可能抛出OOME，所以这里需要捕获OutOfMemoryError，
>                       // 避免因为OOME而导致ReferenceHandler进程静默退出
>                       try {
>                           try {
>                               lock.wait();
>                           } catch (OutOfMemoryError x) { }
>                       } catch (InterruptedException x) { }
>                       continue;
>                   }
>               }
>   
>               // 如果Reference是Cleaner，调用其clean方法
>               // 这与Cleaner机制有关系，不在此文的讨论访问
>               if (r instanceof Cleaner) {
>                   ((Cleaner)r).clean();
>                   continue;
>               }
>   
>               // 把Reference添加到关联的ReferenceQueue中
>               // 如果Reference构造时没有关联ReferenceQueue，会关联ReferenceQueue.NULL，这里就不会进行入队操作了
>               ReferenceQueue<Object> q = r.queue;
>               if (q != ReferenceQueue.NULL) q.enqueue(r);
>           }
>       }
>   }
>   ```
>
>   `ReferenceHandler`线程是在Reference的static块中启动的：
>
>   ```java
>   static {
>       // 获取system ThreadGroup
>       ThreadGroup tg = Thread.currentThread().getThreadGroup();
>       for (ThreadGroup tgn = tg;
>            tgn != null;
>            tg = tgn, tgn = tg.getParent());
>       Thread handler = new ReferenceHandler(tg, "Reference Handler");
>   
>       // ReferenceHandler线程有最高优先级
>       handler.setPriority(Thread.MAX_PRIORITY);
>       handler.setDaemon(true);
>       handler.start();
>   }
>   ```
>
>
>   综上，`ReferenceHandler`是一个最高优先级的线程，其逻辑是从`Pending-Reference`链表中取出Reference，添加到其关联的`Reference-Queue`中。





> + ReferenceQueue
>
>   `Reference-Queue`也是一个链表：
>
>   ```java
>   public class ReferenceQueue<T> {
>       private volatile Reference<? extends T> head = null;
>       // ...
>   }
>   ```
>
>   ```java
>   // ReferenceQueue中的这个锁用于保护链表队列在多线程环境下的正确性
>   static private class Lock { };
>   private Lock lock = new Lock();
>   
>   boolean enqueue(Reference<? extends T> r) { /* Called only by Reference class */
>       synchronized (lock) {
>   		// 判断Reference是否需要入队
>           ReferenceQueue<?> queue = r.queue;
>           if ((queue == NULL) || (queue == ENQUEUED)) {
>               return false;
>           }
>           assert queue == this;
>           
>           // Reference入队后，其queue变量设置为ENQUEUED
>           r.queue = ENQUEUED;
>           // Reference的next变量指向ReferenceQueue中下一个元素
>           r.next = (head == null) ? r : head;
>           head = r;
>           queueLength++;
>           if (r instanceof FinalReference) {
>               sun.misc.VM.addFinalRefCount(1);
>           }
>           lock.notifyAll();
>           return true;
>       }
>   }
>   ```
>
>   通过上面的代码，可以知道`java.lang.ref.Reference#next`的用途了：



### 5. 总结

![Reference](http://imushan.com/img/image-20180819185503111.png)



<img src="https://pic2.zhimg.com/80/v2-d3eef59cdcf0b41318e753da7a787d25_1440w.jpg" alt="或者再简陋一点的图" style="zoom:150%;" />
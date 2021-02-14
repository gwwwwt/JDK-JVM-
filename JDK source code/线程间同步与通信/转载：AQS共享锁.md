### 共享锁的获取与释放（转载）

> 转载自：https://www.itqiankun.com/article/1190000016447307/7000

## 前言

由于AQS对于共享锁与独占锁的实现框架比较类似，因此如果你搞定了前面的独占锁模式，则共享锁也就很容易弄懂了。

[系列文章目录](https://itqiankun.com/article/1190000016058789)

## 共享锁与独占锁的区别

共享锁与独占锁最大的区别在于，独占锁是**独占的，排他的**，因此在独占锁中有一个`exclusiveOwnerThread`属性，用来记录当前持有锁的线程。**当独占锁已经被某个线程持有时，其他线程只能等待它被释放后，才能去争锁，并且同一时刻只有一个线程能争锁成功。**

而对于共享锁而言，由于锁是可以被共享的，因此**它可以被多个线程同时持有**。换句话说，如果一个线程成功获取了共享锁，那么其他等待在这个共享锁上的线程就也可以尝试去获取锁，并且极有可能获取成功。

共享锁的实现和独占锁是对应的，我们可以从下面这张表中看出：

| 独占锁                                      | 共享锁                                            |
| ------------------------------------------- | ------------------------------------------------- |
| tryAcquire(int arg)                         | tryAcquireShared(int arg)                         |
| tryAcquireNanos(int arg, long nanosTimeout) | tryAcquireSharedNanos(int arg, long nanosTimeout) |
| acquire(int arg)                            | acquireShared(int arg)                            |
| acquireQueued(final Node node, int arg)     | doAcquireShared(int arg)                          |
| acquireInterruptibly(int arg)               | acquireSharedInterruptibly(int arg)               |
| doAcquireInterruptibly(int arg)             | doAcquireSharedInterruptibly(int arg)             |
| doAcquireNanos(int arg, long nanosTimeout)  | doAcquireSharedNanos(int arg, long nanosTimeout)  |
| release(int arg)                            | releaseShared(int arg)                            |
| tryRelease(int arg)                         | tryReleaseShared(int arg)                         |
| -                                           | doReleaseShared()                                 |

可以看出，除了最后一个属于共享锁的`doReleaseShared()`方法没有对应外，其他的方法，独占锁和共享锁都是一一对应的。

事实上，其实与`doReleaseShared()`对应的独占锁的方法应当是`unparkSuccessor(h)`，只是`doReleaseShared()`逻辑不仅仅包含了`unparkSuccessor(h)`，还包含了其他操作，这一点我们下面分析源码的时候再看。

另外，尤其需要注意的是，在独占锁模式中，我们只有在获取了独占锁的节点释放锁时，才会唤醒后继节点——这是合理的，因为独占锁只能被一个线程持有，如果它还没有被释放，就没有必要去唤醒它的后继节点。

然而，在共享锁模式下，当一个节点获取到了共享锁，我们在获取成功后就可以唤醒后继节点了，而不需要等到该节点释放锁的时候，这是因为共享锁可以被多个线程同时持有，一个锁获取到了，则后继的节点都可以直接来获取。因此，**在共享锁模式下，在获取锁和释放锁结束时，都会唤醒后继节点。** 这一点也是`doReleaseShared()`方法与`unparkSuccessor(h)`方法无法直接对应的根本原因所在。

## 共享锁的获取

```java
    public final void acquireShared(int arg) {
        if (tryAcquireShared(arg) < 0)
            doAcquireShared(arg);
    }
```

我们拿它和独占锁模式对比一下：

```java
    public final void acquire(int arg) {
        if (!tryAcquire(arg) && acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
            selfInterrupt();
    }
```

这两者的结构看上去似乎有点差别，但事实上是一样的，只不过是共享锁模式下，将与`addWaiter(Node.EXCLUSIVE)`对应的`addWaiter(Node.SHARED)`，以及`selfInterrupt()`操作全部移到了`doAcquireShared`方法内部，这一点我们在下面分析`doAcquireShared`方法时就一目了然了。

不过这里先插一句，相对于独占的锁的`tryAcquire(int arg)`返回boolean类型的值，共享锁的`tryAcquireShared(int acquires)`返回的是一个整型值：

- 如果该值小于0，则代表当前线程获取共享锁失败
- 如果该值大于0，则代表当前线程获取共享锁成功，并且接下来其他线程尝试获取共享锁的行为很可能成功
- 如果该值等于0，则代表当前线程获取共享锁成功，但是接下来其他线程尝试获取共享锁的行为会失败

因此，只要该返回值大于等于0，就表示获取共享锁成功。

`acquireShared`中的`tryAcquireShared`方法由具体的子类负责实现，这里我们暂且不表。

接下来我们看看`doAcquireShared`方法，它对应于独占锁的`acquireQueued`，两者其实很类似，我们把它们相同的部分注释掉，只看不同的部分：

```java
    private void doAcquireShared(int arg) {
        final Node node = addWaiter(Node.SHARED);
        /*boolean failed = true;
        try {
            boolean interrupted = false;
            for (;;) {
                final Node p = node.predecessor();*/
                if (p == head) {
                    int r = tryAcquireShared(arg);
                    if (r >= 0) {
                        setHeadAndPropagate(node, r);
                        p.next = null; // help GC
                        if (interrupted)
                            selfInterrupt();
                        failed = false;
                        return;
                    }
                }
                /*if (shouldParkAfterFailedAcquire(p, node) &&
                    parkAndCheckInterrupt())
                    interrupted = true;
            }
        } finally {
            if (failed)
                cancelAcquire(node);
        }*/
    }
```

关于上面的if部分，独占锁对应的`acquireQueued`方法为：

```java
    if (p == head && tryAcquire(arg)) {
        setHead(node);
        p.next = null; // help GC
        failed = false;
        return interrupted;
    }
```

因此，综合来看，这两者的逻辑仅有两处不同：

1. `addWaiter(Node.EXCLUSIVE)` -> `addWaiter(Node.SHARED)`
2. `setHead(node)` -> `setHeadAndPropagate(node, r)`

这里第一点不同就是独占锁的`acquireQueued`调用的是`addWaiter(Node.EXCLUSIVE)`，而共享锁调用的是`addWaiter(Node.SHARED)`，表明了该节点处于共享模式，这两种模式的定义为：

```java
    /** Marker to indicate a node is waiting in shared mode */
    static final Node SHARED = new Node();
    /** Marker to indicate a node is waiting in exclusive mode */
    static final Node EXCLUSIVE = null;
```

该模式被赋值给了节点的`nextWaiter`属性：

```java
    Node(Thread thread, Node mode) {     // Used by addWaiter
        this.nextWaiter = mode;
        this.thread = thread;
    }
```

我们知道，在条件队列中，`nextWaiter`是指向条件队列中的下一个节点的，它将条件队列中的节点串起来，构成了单链表。但是在`sync queue`队列中，我们只用`prev`,`next`属性来串联节点，形成双向链表，`nextWaiter`属性在这里只起到一个**标记作用**，不会串联节点，这里不要被`Node SHARED = new Node()`所指向的空节点迷惑，这个空节点并不属于`sync queue`，不代表任何线程，它只起到标记作用，仅仅用作判断节点是否处于共享模式的依据：

```java
    // Node#isShard()
    final boolean isShared() {
        return nextWaiter == SHARED;
    }
```

这里的第二点不同就在于获取锁成功后的行为，对于独占锁而言，是直接调用了`setHead(node)`方法，而共享锁调用的是`setHeadAndPropagate(node, r)`：

```java
    private void setHeadAndPropagate(Node node, int propagate) {
        Node h = head; // Record old head for check below
        setHead(node);
        if (propagate > 0 || h == null || h.waitStatus < 0 || (h = head) == null || h.waitStatus < 0) {
            Node s = node.next;
            if (s == null || s.isShared())
                doReleaseShared();
        }
    }
```

在该方法内部我们不仅调用了`setHead(node)`，还在一定条件下调用了`doReleaseShared()`来唤醒后继的节点。这是因为**在共享锁模式下，锁可以被多个线程所共同持有，既然当前线程已经拿到共享锁了，那么就可以直接通知后继节点来拿锁，而不必等待锁被释放的时候再通知。**

关于这个`doReleaseShared`方法，我们到下面分析锁释放的时候再看。

## 共享锁的释放

我们使用`releaseShared(int arg)`方法来释放共享锁：

```java
    public final boolean releaseShared(int arg) {
        if (tryReleaseShared(arg)) {
            doReleaseShared();
            return true;
        }
        return false;
    }
```

该方法对应于独占锁的`release(int arg)`方法：

```java
    public final boolean release(int arg) {
        if (tryRelease(arg)) {
            Node h = head;
            if (h != null && h.waitStatus != 0)
                unparkSuccessor(h);
            return true;
        }
        return false;
    }
```

在独占锁模式下，由于头节点就是持有独占锁的节点，在它释放独占锁后，如果发现自己的waitStatus不为0，则它将负责唤醒它的后继节点。

在共享锁模式下，头节点就是持有共享锁的节点，在它释放共享锁后，它也应该唤醒它的后继节点，但是值得注意的是，我们在之前的`setHeadAndPropagate`方法中可能已经调用过该方法了，也就是说**它可能会被同一个头节点调用两次**，也有可能在我们从`releaseShared`方法中调用它时，当前的头节点已经易主了，下面我们就来详细看看这个方法：

```java
    private void doReleaseShared() {
        for (;;) {
            Node h = head;
            if (h != null && h != tail) {
                int ws = h.waitStatus;
                if (ws == Node.SIGNAL) {
                    if (!compareAndSetWaitStatus(h, Node.SIGNAL, 0))
                        continue;            // loop to recheck cases
                    unparkSuccessor(h);
                }
                else if (ws == 0 &&
                         !compareAndSetWaitStatus(h, 0, Node.PROPAGATE))
                    continue;                // loop on failed CAS
            }
            if (h == head)                   // loop if head changed
                break;
        }
    }
```

该方法可能是共享锁模式最难理解的方法了，在看该方法时，我们需要明确以下几个问题：

**(1) 该方法有几处调用？**

该方法有两处调用，一处在`acquireShared`方法的末尾，当线程成功获取到共享锁后，在一定条件下调用该方法；一处在`releaseShared`方法中，当线程释放共享锁的时候调用。

**(2) 调用该方法的线程是谁？**

在独占锁中，只有获取了锁的线程才能调用release释放锁，因此调用unparkSuccessor(h)唤醒后继节点的必然是持有锁的线程，该线程可看做是当前的头节点(虽然在setHead方法中已经将头节点的thread属性设为了null，但是这个头节点曾经代表的就是这个线程)

在共享锁中，持有共享锁的线程可以有多个，这些线程都可以调用`releaseShared`方法释放锁；而这些线程想要获得共享锁，则它们必然**曾经成为过头节点，或者就是现在的头节点**。因此，如果是在`releaseShared`方法中调用的`doReleaseShared`，可能此时调用方法的线程已经不是头节点所代表的线程了，头节点可能已经被易主好几次了。

**(3) 调用该方法的目的是什么？**

无论是在`acquireShared`中调用，还是在`releaseShared`方法中调用，该方法的目的都是在当前共享锁是可获取的状态时，**唤醒head节点的下一个节点**。这一点看上去和独占锁似乎一样，但是它们的一个重要的差别是——在共享锁中，当头节点发生变化时，是会回到循环中再**立即**唤醒head节点的下一个节点的。也就是说，在当前节点完成唤醒后继节点的任务之后将要退出时，如果发现被唤醒后继节点已经成为了新的头节点，则会立即触发**唤醒head节点的下一个节点**的操作，如此周而复始。

**(4) 退出该方法的条件是什么**

该方法是一个自旋操作(`for(;;)`)，退出该方法的唯一办法是走最后的break语句：

```java
    if (h == head)   // loop if head changed
        break;
```

即，只有在当前head没有易主时，才会退出，否则继续循环。

这个怎么理解呢？

为了说明问题，这里我们假设目前sync queue队列中依次排列有

> dummy node -> A -> B -> C -> D

现在假设A已经拿到了共享锁，则它将成为新的dummy node，

> dummy node (A) -> B -> C -> D

此时，A线程会调用doReleaseShared，我们写做`doReleaseShared[A]`，在该方法中将唤醒后继的节点B，它很快获得了共享锁，成为了新的头节点：

> dummy node (B) -> C -> D

此时，B线程也会调用doReleaseShared，我们写做`doReleaseShared[B]`，在该方法中将唤醒后继的节点C，但是别忘了，在`doReleaseShared[B]`调用的时候，`doReleaseShared[A]`还没运行结束呢，当它运行到`if(h == head)`时，发现头节点现在已经变了，所以它将继续回到for循环中，与此同时，`doReleaseShared[B]`也没闲着，它在执行过程中也进入到了for循环中。。。

由此可见，我们这里形成了一个doReleaseShared的“**调用风暴**”，大量的线程在同时执行doReleaseShared，这极大地加速了唤醒后继节点的速度，提升了效率，同时该方法内部的CAS操作又保证了多个线程同时唤醒一个节点时，只有一个线程能操作成功。

那如果这里`doReleaseShared[A]`执行结束时，节点B还没有成为新的头节点时，`doReleaseShared[A]`方法不就退出了吗？是的，但即使这样也没有关系，因为它已经成功唤醒了线程B，即使`doReleaseShared[A]`退出了，当B线程成为新的头节点时，`doReleaseShared[B]`就开始执行了，它也会负责唤醒后继节点的，这样即使变成这种每个节点只唤醒自己后继节点的模式，从功能上讲，最终也可以实现唤醒所有等待共享锁的节点的目的，只是效率上没有之前的“调用风暴”快。

由此我们知道，这里的“调用风暴”事实上是一个优化操作，因为在我们执行到该方法的末尾的时候，`unparkSuccessor`基本上已经被调用过了，而由于现在是共享锁模式，所以**被唤醒的后继节点**极有可能已经获取到了共享锁，成为了新的head节点，当它成为新的head节点后，它可能还是要在`setHeadAndPropagate`方法中调用`doReleaseShared`唤醒它的后继节点。

明确了上面几个问题后，我们再来详细分析这个方法，它最重要的部分就是下面这两个if语句：

```java
    if (ws == Node.SIGNAL) {
        if (!compareAndSetWaitStatus(h, Node.SIGNAL, 0))
            continue;            // loop to recheck cases
        unparkSuccessor(h);
    }
    else if (ws == 0 &&
             !compareAndSetWaitStatus(h, 0, Node.PROPAGATE))
        continue;                // loop on failed CAS
```

第一个if很好理解，如果当前ws值为Node.SIGNAL，则说明后继节点需要唤醒，这里采用CAS操作先将Node.SIGNAL状态改为0，这是因为前面讲过，可能有大量的doReleaseShared方法在同时执行，我们只需要其中一个执行`unparkSuccessor(h)`操作就行了，这里通过CAS操作保证了`unparkSuccessor(h)`只被执行一次。

比较难理解的是第二个else if，首先我们要弄清楚ws啥时候为0，一种是上面的`compareAndSetWaitStatus(h, Node.SIGNAL, 0)`会导致ws为0，但是很明显，如果是因为这个原因，则它是不会进入到else if语句块的。所以这里的ws为0是指**当前队列的最后一个节点成为了头节点**。为什么是最后一个节点呢，因为每次新的节点加进来，在挂起前一定会将自己的前驱节点的waitStatus修改成Node.SIGNAL的。(对这一点不理解的详细看[这里](https://itqiankun.com/article/1190000015739343))

其次，`compareAndSetWaitStatus(h, 0, Node.PROPAGATE)`这个操作什么时候会失败？既然这个操作失败，说明就在执行这个操作的瞬间，ws此时已经不为0了，说明有新的节点入队了，ws的值被改为了Node.SIGNAL，此时我们将调用`continue`，在下次循环中直接将这个刚刚新入队但准备挂起的线程唤醒。

其实，如果我们再结合外部的整体条件，就很容易理解这种情况所针对的场景，不要忘了，进入上面这段还有一个条件是

```java
    if (h != null && h != tail)
```

它处于最外层：

```java
    private void doReleaseShared() {
        for (;;) {
            Node h = head;
            if (h != null && h != tail) { // 注意这里说明了队列至少有两个节点
                int ws = h.waitStatus;
                if (ws == Node.SIGNAL) {
                    if (!compareAndSetWaitStatus(h, Node.SIGNAL, 0))
                        continue;            
                    unparkSuccessor(h);
                }
                else if (ws == 0 && !compareAndSetWaitStatus(h, 0, Node.PROPAGATE))
                    continue;               
            }
            if (h == head)
                break;
        }
    }
```

这个条件意味着，队列中至少有两个节点。

结合上面的分析，我们可以看出，这个

```java
    else if (ws == 0 && !compareAndSetWaitStatus(h, 0, Node.PROPAGATE))
```

描述了一个极其严苛且短暂的状态：

1. 首先，大前提是队列里至少有两个节点
2. 其次，要执行到`else if`语句，说明我们跳过了前面的if条件，说明头节点是刚刚成为头节点的，它的waitStatus值还为0，尾节点是在这之后刚刚加进来的，它需要执行`shouldParkAfterFailedAcquire`，将它的前驱节点（即头节点）的waitStatus值修改为`Node.SIGNAL`，**但是目前这个修改操作还没有来的及执行**。这种情况使我们得以进入else if的前半部分`else if (ws == 0 &&`
3. 紧接着，要满足`!compareAndSetWaitStatus(h, 0, Node.PROPAGATE)`这一条件，说明此时头节点的`waitStatus`已经不是0了，这说明之前那个没有来得及执行的 **在`shouldParkAfterFailedAcquire`将前驱节点的的waitStatus值修改为`Node.SIGNAL`的操作**现在执行完了。

由此可见，`else if` 的 `&&` 连接了两个不一致的状态，分别对应了`shouldParkAfterFailedAcquire`的`compareAndSetWaitStatus(pred, ws, Node.SIGNAL)`执行成功前和执行成功后，因为`doReleaseShared`和
`shouldParkAfterFailedAcquire`是可以并发执行的，所以这一条件是有可能满足的，只是满足的条件非常严苛，可能只是一瞬间的事。

这里不得不说，如果以上的分析没有错的话，那作者对于AQS性能的优化已经到了“令人发指”的地步！！！虽说这种短暂的瞬间确实存在，也确实有必要重新回到for循环中再次去唤醒后继节点，但是这种优化也太太太～～～过于精细了吧！

我们来看看如果不加入这个精细的控制条件有什么后果呢？

这里我们复习一下新节点入队的过程，前面说过，在发现新节点的前驱不是head节点的时候，它将调用`shouldParkAfterFailedAcquire`：

```java
    private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
        int ws = pred.waitStatus;
        if (ws == Node.SIGNAL)
            /*
             * This node has already set status asking a release
             * to signal it, so it can safely park.
             */
            return true;
        if (ws > 0) {
            /*
             * Predecessor was cancelled. Skip over predecessors and
             * indicate retry.
             */
            do {
                node.prev = pred = pred.prev;
            } while (pred.waitStatus > 0);
            pred.next = node;
        } else {
            /*
             * waitStatus must be 0 or PROPAGATE.  Indicate that we
             * need a signal, but don't park yet.  Caller will need to
             * retry to make sure it cannot acquire before parking.
             */
            compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
        }
        return false;
    }
```

由于前驱节点的ws值现在还为0，新节点将会把它改为Node.SIGNAL，

但修改后，该方法返回的是false，也就是说线程不会立即挂起，而是回到上层再尝试一次抢锁：

```java
    private void doAcquireShared(int arg) {
        final Node node = addWaiter(Node.SHARED);
        boolean failed = true;
        try {
            boolean interrupted = false;
            for (;;) {
                final Node p = node.predecessor();
                if (p == head) {
                    int r = tryAcquireShared(arg);
                    if (r >= 0) {
                        setHeadAndPropagate(node, r);
                        p.next = null; // help GC
                        if (interrupted)
                            selfInterrupt();
                        failed = false;
                        return;
                    }
                }
                // shouldParkAfterFailedAcquire的返回处
                if (shouldParkAfterFailedAcquire(p, node) &&
                    parkAndCheckInterrupt())
                    interrupted = true;
            }
        } finally {
            if (failed)
                cancelAcquire(node);
        }
    }
```

当我们再次回到`for(;;)`循环中，由于此时当前节点的前驱节点已经成为了新的head，所以它可以参与抢锁，由于它抢的是共享锁，所以大概率它是抢的到的，所以极有可能它不会被挂起。这有可能导致在上面的`doReleaseShared`调用`unparkSuccessor`方法`unpark`了一个并没有被`park`的线程。然而，这一操作是被允许的，当我们`unpark`一个并没有被`park`的线程时，该线程在下一次调用`park`方法时就不会被挂起，而这一行为是符合我们的场景的——因为当前的共享锁处于可获取的状态，后继的线程应该直接来获取锁，不应该被挂起。

```java
    else if (ws == 0 && !compareAndSetWaitStatus(h, 0, Node.PROPAGATE))
        continue;  // loop on failed CAS
```

这一段其实也可以省略，当然有了这一段肯定会加速唤醒后继节点的过程，作者针对上面那种极其短暂的情况进行了优化可以说是和它之前“调用风暴”的设计一脉相承，可能也正是由于作者对于性能的极致追求才使得AQS如此之优秀吧。

## 总结

- 共享锁的调用框架和独占锁很相似，它们最大的不同在于获取锁的逻辑——共享锁可以被多个线程同时持有，而独占锁同一时刻只能被一个线程持有。
- 由于共享锁同一时刻可以被多个线程持有，因此当头节点获取到共享锁时，可以立即唤醒后继节点来争锁，而不必等到释放锁的时候。因此，共享锁触发唤醒后继节点的行为可能有两处，一处在当前节点成功获得共享锁后，一处在当前节点释放共享锁后。
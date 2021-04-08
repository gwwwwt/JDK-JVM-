# 线程中断interrupt

> 转载自: [Thread类源码解读(3)——线程中断interrupt](https://www.itqiankun.com/article/1190000016083002)

## 前言

线程中断是一个很重要的概念，通常，取消一个任务的执行，最好的，同时也是最合理的方法，就是通过中断。

本篇我们主要还是通过源码分析来看看中断的概念。

本文的源码基于JDK1.8

## Interrupt status & InterruptedException

java线程的中断机制为我们提供了一个契机，使被中断的线程能够有机会从当前的任务中跳脱出来。而中断机制的最核心的两个概念就是`interrupt status` 和 `InterruptedException`。

java中对于中断的大部分操作无外乎以下两点:

1. 设置或者清除中断标志位
2. 抛出`InterruptedException`

### interrupt status

在java中，每一个线程都有一个中断标志位，表征了当前线程是否处于被中断状态，我们可以把这个标识位理解成一个boolean类型的变量，当我们中断一个线程时，将该标识位设为true，当我们清除中断状态时，将其设置为false, 其伪代码如下:

(注意，本文的伪代码部分是我个人所写，并不权威，只是帮助我自己理解写的)

```java
    // 注意，这是伪代码！！！
    // 注意，这是伪代码！！！
    // 注意，这是伪代码！！！
    public class Thread implements Runnable {
        private boolean interruptFlag; // 中断标志位
        public boolean getInterruptFlag() {
            return this.interruptFlag;
        }
        public void setInterruptFlag(boolean flag) {
            this.interruptFlag = flag;
        }
    }
```

然而，**在Thread线程类里面，并没有类似中断标志位的属性，但是提供了获取中断标志位的接口:**

```java
    /**
     * Tests if some Thread has been interrupted.  The interrupted state
     * is reset or not based on the value of ClearInterrupted that is
     * passed.
     */
    private native boolean isInterrupted(boolean ClearInterrupted);
```

这是一个native方法，同时也是一个**private方法**，该方法除了能够返回当前线程的中断状态，还能根据`ClearInterrupted`参数来决定要不要重置中断标志位(reset操作相当于上面的`interruptFlag = false`)。

Thread类提供了两个public方法来使用该native方法:

```java
    public boolean isInterrupted() {
        return isInterrupted(false);
    }
    public static boolean interrupted() {
        return currentThread().isInterrupted(true);
    }
```

其中`isInterrupted`调用了`isInterrupted(false)`, `ClearInterrupted`参数为`false`, 说明它仅仅返回线程实例的中断状态，但是不会对现有的中断状态做任何改变，伪代码可以是:

```java
    // 注意，这是伪代码！！！
    // 注意，这是伪代码！！！
    // 注意，这是伪代码！！！
    public boolean isInterrupted() {
        return interruptFlag; //直接返回Thread实例的中断状态
    }
```

而`interrupted`是一个**静态方法**，所以它可以由Thread类直接调用，自然就是作用于**当前正在执行的线程**，所以函数内部使用了currentThread()方法，与`isInterrupted()`方法不同的是，它的`ClearInterrupted`参数为`true`，在返回线程中断状态的同时，重置了中断标识位，伪代码可以是:

```
    // 注意，这是伪代码！！！
    // 注意，这是伪代码！！！
    // 注意，这是伪代码！！！
    public static boolean interrupted() {
        Thread current = Thread.currentThread(); // 获取当前正在执行的线程
        boolean interruptFlag = current.getInterruptFlag(); // 获取线程的中断状态
        current.setInterruptFlag(false); // 清除线程的中断状态
        return interruptFlag; //返回线程的中断状态
    }
```

可见，`isInterrupted()` 和 `interrupted()` 方法只涉及到中断状态的查询，最多是多加一步重置中断状态，并不牵涉到`InterruptedException`。

不过值得一提的是，**在我们能使用到的public方法中，`interrupted()`是我们清除中断的唯一方法。**

### InterruptedException

我们直接来看的源码:

```java
    /**
     * Thrown when a thread is waiting, sleeping, or otherwise occupied,
     * and the thread is interrupted, either before or during the activity.
     * Occasionally a method may wish to test whether the current
     * thread has been interrupted, and if so, to immediately throw
     * this exception.  The following code can be used to achieve
     * this effect:
     * <pre>
     *  if (Thread.interrupted())  // Clears interrupted status!
     *      throw new InterruptedException();
     * </pre>
     *
     * @author  Frank Yellin
     * @see     java.lang.Object#wait()
     * @see     java.lang.Object#wait(long)
     * @see     java.lang.Object#wait(long, int)
     * @see     java.lang.Thread#sleep(long)
     * @see     java.lang.Thread#interrupt()
     * @see     java.lang.Thread#interrupted()
     * @since   JDK1.0
     */
    public class InterruptedException extends Exception {
        private static final long serialVersionUID = 6700697376100628473L;
        /**
         * Constructs an <code>InterruptedException</code> with no detail  message.
         */
        public InterruptedException() {
            super();
        }
        /**
         * Constructs an <code>InterruptedException</code> with the
         * specified detail message.
         *
         * @param   s   the detail message.
         */
        public InterruptedException(String s) {
            super(s);
        }
    }
```

上面的注释是说，在线程处于“waiting, sleeping”甚至是正在运行的过程中，如果被中断了，就可以抛出该异常，我们先来回顾一下我们前面遇到过的抛出`InterruptedException`异常的例子:

(1) wait(long timeout)方法中的InterruptedException

```java
    /* 
     *
     * @param      timeout   the maximum time to wait in milliseconds.
     * @throws  IllegalArgumentException      if the value of timeout is
     *               negative.
     * @throws  IllegalMonitorStateException  if the current thread is not
     *               the owner of the object's monitor.
     * @throws  InterruptedException if any thread interrupted the
     *             current thread before or while the current thread
     *             was waiting for a notification.  The <i>interrupted
     *             status</i> of the current thread is cleared when
     *             this exception is thrown.
     * @see        java.lang.Object#notify()
     * @see        java.lang.Object#notifyAll()
     */
    public final native void wait(long timeout) throws InterruptedException;
```

该方法的注释中提到，如果在有别的线程在当前线程进入waiting状态之前或者已经进入waiting状态之后中断了当前线程，该方法就会抛出`InterruptedException`，同时，异常抛出后，当前线程的中断状态也会被清除。

(2) sleep(long millis)方法中的InterruptedException

```java
    /* @param  millis
     *         the length of time to sleep in milliseconds
     *
     * @throws  IllegalArgumentException
     *          if the value of {@code millis} is negative
     *
     * @throws  InterruptedException
     *          if any thread has interrupted the current thread. The
     *          <i>interrupted status</i> of the current thread is
     *          cleared when this exception is thrown.
     */
    public static native void sleep(long millis) throws InterruptedException;
```

与上面的wait方法一致，如果当前线程被中断了，sleep方法会抛出InterruptedException，并且清除中断状态。

如果有其他方法直接或间接的调用了这两个方法，那他们自然也会在线程被中断的时候抛出InterruptedException，并且清除中断状态。例如:

- wait()
- wait(long timeout, int nanos)
- sleep(long millis, int nanos)
- join()
- join(long millis)
- join(long millis, int nanos)

这里值得注意的是，虽然这些方法会抛出InterruptedException，但是并不会终止当前线程的执行，当前线程可以选择忽略这个异常。

也就是说，无论是设置`interrupt status` 还是抛出`InterruptedException`，它们都是给当前线程的**建议**，当前线程可以选择采纳或者不采纳，它们并不会影响当前线程的执行。

至于在收到这些中断的建议后，当前线程要怎么处理，也完全取决于当前线程。

## interrupt

上面我们说了怎么检查(以及清除)一个线程的中断状态，提到当一个线程被中断后，有一些方法会抛出InterruptedException。

下面我们就来看看怎么中断一个线程。

要中断一个线程，只需调用该线程的`interrupt`方法，其源码如下:

```java
    /**
     * Interrupts this thread.
     *
     * <p> Unless the current thread is interrupting itself, which is
     * always permitted, the {@link #checkAccess() checkAccess} method
     * of this thread is invoked, which may cause a {@link
     * SecurityException} to be thrown.
     *
     * <p> If this thread is blocked in an invocation of the {@link
     * Object#wait() wait()}, {@link Object#wait(long) wait(long)}, or {@link
     * Object#wait(long, int) wait(long, int)} methods of the {@link Object}
     * class, or of the {@link #join()}, {@link #join(long)}, {@link
     * #join(long, int)}, {@link #sleep(long)}, or {@link #sleep(long, int)},
     * methods of this class, then its interrupt status will be cleared and it
     * will receive an {@link InterruptedException}.
     *
     * <p> If this thread is blocked in an I/O operation upon an {@link
     * java.nio.channels.InterruptibleChannel InterruptibleChannel}
     * then the channel will be closed, the thread's interrupt
     * status will be set, and the thread will receive a {@link
     * java.nio.channels.ClosedByInterruptException}.
     *
     * <p> If this thread is blocked in a {@link java.nio.channels.Selector}
     * then the thread's interrupt status will be set and it will return
     * immediately from the selection operation, possibly with a non-zero
     * value, just as if the selector's {@link
     * java.nio.channels.Selector#wakeup wakeup} method were invoked.
     *
     * <p> If none of the previous conditions hold then this thread's interrupt
     * status will be set. </p>
     *
     * <p> Interrupting a thread that is not alive need not have any effect.
     *
     * @throws  SecurityException
     *          if the current thread cannot modify this thread
     *
     * @revised 6.0
     * @spec JSR-51
     */
    public void interrupt() {
        if (this != Thread.currentThread())
            checkAccess();
        synchronized (blockerLock) {
            Interruptible b = blocker;
            if (b != null) {
                interrupt0();           // Just to set the interrupt flag
                b.interrupt(this);
                return;
            }
        }
        interrupt0();
    }
```

上面的注释很长，我们一段一段来看:

```java
    /**
     * Interrupts this thread.
     *
     * <p> Unless the current thread is interrupting itself, which is
     * always permitted, the {@link #checkAccess() checkAccess} method
     * of this thread is invoked, which may cause a {@link
     * SecurityException} to be thrown.
     ...
     */
```

上面这段首先说明了这个函数的目的是中断这个线程，这个`this thread`，当然指的就是该方法所属的线程对象所代表的线程。

接着说明了，一个线程总是被允许中断自己，但是我们如果想要在一个线程中中断另一个线程的执行，就需要先通过checkAccess()检查权限。这有可能抛出SecurityException异常, 这段话用代码体现为:

```java
    if (this != Thread.currentThread())
        checkAccess();
```

我们接着往下看:

```java
    /*
     ...
     * <p> If this thread is blocked in an invocation of the {@link
     * Object#wait() wait()}, {@link Object#wait(long) wait(long)}, or {@link
     * Object#wait(long, int) wait(long, int)} methods of the {@link Object}
     * class, or of the {@link #join()}, {@link #join(long)}, {@link
     * #join(long, int)}, {@link #sleep(long)}, or {@link #sleep(long, int)},
     * methods of this class, then its interrupt status will be cleared and it
     * will receive an {@link InterruptedException}.
     ...
     */
```

上面这段是说，如果线程因为以下方法的调用而处于阻塞中，那么(调用了interrupt方法之后)，线程的中断标志会被清除，并且收到一个InterruptedException:

- Object的方法
  - wait()
  - wait(long)
  - wait(long, int)
- Thread的方法
  - join()
  - join(long)
  - join(long, int)
  - sleep(long)
  - sleep(long, int)

关于这一点，我们上面在分析InterruptedException的时候已经分析过了。

这里插一句，由于上面这些方法在抛出InterruptedException异常后，会**同时清除中断标识位**，因此当我们此时不想或者无法传递InterruptedException异常，也不对该异常做任何处理时，我们最好通过再次调用interrupt来恢复中断的状态，以供上层调用者处理，这一点，我们在[逐行分析AQS源码(二): 锁的释放](https://itqiankun.com/article/1190000015752512)的最后就说明过这种用法。

接下来的两段注释是关于NIO的，我们暂时不看，直接看最后两段:

```java
    /*
     ...
     * <p> If none of the previous conditions hold then this thread's interrupt
     * status will be set. </p>
     *
     * <p> Interrupting a thread that is not alive need not have any effect.
     */
```

这段话是说:

- 如果线程没有因为上面的函数调用而进入阻塞状态的话，那么中断这个线程仅仅会设置它的中断标志位(而不会抛出InterruptedException)
- 中断一个已经终止的线程不会有任何影响。

注释看完了之后我们再来看代码部分，其实代码部分很简单，中间那段同步代码块是和NIO有关的，我们可以暂时不管，整个方法的核心调用就是`interrupt0()`方法，而它是一个native方法:

```java
    private native void interrupt0();
```

这个方法所做的事情很简单:

> Just to set the interrupt flag

所以，至此我们明白了**，所谓“中断一个线程”，其实并不是让一个线程停止运行，仅仅是将线程的中断标志设为`true`, 或者在某些特定情况下抛出一个InterruptedException，它并不会直接将一个线程停掉，在被中断的线程的角度看来，仅仅是自己的中断标志位被设为`true`了，或者自己所执行的代码中抛出了一个`InterruptedException`异常，仅此而已。**

## 终止一个线程

既然上面我们提到了，中断一个线程并不会使得该线程停止执行，那么我们该怎样终止一个线程的执行呢。早期的java中提供了stop()方法来停止一个线程，但是这个方法是不安全的，所以已经被废弃了。现在终止一个线程，基本上只能靠“曲线救国”式的中断来实现。

### 终止处于阻塞状态的线程

前面我们说过，当一个线程因为调用wait,sleep,join方法而进入阻塞状态后，若在这时中断这个线程，则这些方法将会抛出`InterruptedException`异常，我们可以利用这个异常，使线程跳出阻塞状态，从而终止线程。

```java
    @Override
    public void run() {
        while(true) {
            try {
                // do some task
                // blocked by calling wait/sleep/join
            } catch (InterruptedException ie) {  
                // 如果该线程被中断，则会抛出InterruptedException异常
                // 我们通过捕获这个异常，使得线程从block状态退出
                break; // 这里使用break, 可以使我们在线程中断后退出死循环，从而终止线程。
            }
        }
    }
```

### 终止处于运行状态的线程

与中断一个处于阻塞状态所不同的是，中断一个处于运行状态的线程只会将该线程的中断标志位设为`true`, 而并不会抛出InterruptedException异常，为了能在运行过程中感知到线程已经被中断了，我们只能通过不断地检查中断标志位来实现:

```java
    @Override
    public void run() {
        while (!isInterrupted()) {
            // do some task...
        }
    }
```

这里，我们每次循环都会先检查中断标志位，只要当前线程被中断了，`isInterrupted()`方法就会返回`true`，从而终止循环。

### 终止一个Alive的线程

上面我们分别介绍了怎样终止一个处于阻塞状态或运行状态的线程，如果我们将这两种方法结合起来，那么就可以同时应对这两种状况，从而能够终止任意一个存活的线程:

```java
    @Override
    public void run() {
        try {
            // 1. isInterrupted() 用于终止一个正在运行的线程。
            while (!isInterrupted()) {
                // 执行任务...
            }
        } catch (InterruptedException ie) {  
            // 2. InterruptedException异常用于终止一个处于阻塞状态的线程
        }
    }
```

不过使用这两者的组合一定要注意，wait,sleep,join等方法抛出InterruptedException有一个副作用: 清除当前的中断标志位，所以不要在异常抛出后不做任何处理，而寄望于用`isInterrupted()`方法来判断，因为中标志位已经被重置了，所以下面这种写法是不对的:

```java
    @Override
    public void run() {
        //isInterrupted() 用于终止一个正在运行的线程。
        while (!isInterrupted()) {
            try {
                    // 执行任务...
                }
            } catch (InterruptedException ie) {  
                // 在这里不做任何处理，仅仅依靠isInterrupted检测异常
            }
        }
    }
```

这个方法中，在catch块中我们检测到异常后没有使用break方法跳出循环，而此时中断状态已经被重置，当我们再去调用`isInterrupted`，依旧会返回`false`, 故线程仍然会在while循环中执行，无法被中断。

## 总结

Java没有提供一种安全直接的方法来停止某个线程，但是提供了中断机制。对于被中断的线程，中断只是一个建议，至于收到这个建议后线程要采取什么措施，完全由线程自己决定。

中断机制的核心在于中断状态和`InterruptedException`异常

中断状态:

- 设置一个中断状态: Thread#interrupt
- 清除一个中断状态: Thread.interrupted

Thread.interrupted方法同时会返回线程原来的中断的状态。
如果仅仅想查看线程当前的中断状态而不清除原来的状态，则应该使用Thread#isInterrupted。

某些阻塞方法在抛出InterruptedException异常后，会同时清除中断状态。若不能对该异常做出处理也无法向上层抛出，则应该通过再次调用interrupt方法恢复中断状态，以供上层处理，通常情况下我们都不应该屏蔽中断请求。

中断异常:

中断异常一般是线程被中断后，在一些block类型的方法(如`wait`,`sleep`,`join`)中抛出。

我们可以使用Thread#interrupt中断一个线程，被中断的线程所受的影响为以下两种之一:

- 若被中断前，该线程处于非阻塞状态，那么该线程的中断状态被设为true, 除此之外，不会发生任何事。
- 若被中断前，该线程处于阻塞状态(调用了wait,sleep,join等方法)，那么该线程将会立即从阻塞状态中退出，并抛出一个InterruptedException异常，同时，该线程的中断状态被设为false, 除此之外，不会发生任何事。

无论是中断状态的改变还是`InterruptedException`被抛出，这些都是当前线程可以感知到的”建议”，如果当前线程选择忽略这些建议(例如简单地catch住异常继续执行)，那么中断机制对于当前线程就没有任何影响，就好像什么也没有发生一样。

所以，中断一个线程，只是传递了请求中断的消息，并不会直接阻止一个线程的运行。
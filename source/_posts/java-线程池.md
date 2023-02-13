---
title: java 线程池
date: 2020-09-04 15:02:03
tags: 
 - Java 
 - 线程池
description: Java线程池相关总结
cover: /blogImg/线程基本状态.png
categories: Java基础
typora-root-url: ../../themes/butterfly/source
---
# 线程的生命周期及基本状态

![线程基本状态](/blogImg/线程基本状态.png)

1. 新建(new)：新创建了一个线程对象。

2. 可运行(runnable)：线程对象创建后，当调用线程对象的 start()方法，该线程处于就绪状态，等待被线程调度选中，获取cpu的使用权。

3. 运行(running)：可运行状态(runnable)的线程获得了cpu时间片（timeslice），执行程序代码。注：就绪状态是进入到运行状态的唯一入口，也就是说，线程要想进入运行状态执行，首先必须处于就绪状态中；

4. 阻塞(block)：处于运行状态中的线程由于某种原因，暂时放弃对 CPU的使用权，停止执行，此时进入阻塞状态，直到其进入到就绪状态，才有机会再次被 CPU 调用以进入到运行状态。

	阻塞的情况分三种：
	(一). 等待阻塞：运行状态中的线程执行 wait()方法，JVM会把该线程放入等待队列(waitting queue)中，使本线程进入到等待阻塞状态；
	(二). 同步阻塞：线程在获取 synchronized 同步锁失败(因为锁被其它线程所占用)，则JVM会把该线程放入锁池(lock pool)中，线程会进入同步阻塞状态；
	(三). 其他阻塞: 通过调用线程的 sleep()或 join()或发出了 I/O 请求时，线程会进入到阻塞状态。当 sleep()状态超时、join()等待线程终止或者超时、或者 I/O 处理完毕时，线程重新转入就绪状态。

5. 死亡(dead)：线程run()、main()方法执行结束，或者因异常退出了run()方法，则该线程结束生命周期。死亡的线程不可再次复生。

# 创建线程的几种方式

- 继承 Thread 类；
- 实现 Runnable 接口；
- 实现 Callable 接口；
- 使用 Executors 工具类创建线程池

## **继承 Thread 类**

1. 定义一个Thread类的子类，重写run方法，将相关逻辑实现，run()方法就是线程要执行的业务逻辑方法
2. 创建自定义的线程子类对象
3. 调用子类实例的star()方法来启动线程

```java
public class MyThread extends Thread {
    @Override
    public void run() {
        System.out.println(Thread.currentThread().getName() + " run()方法正在执行...");
    }
}
public class TheadTest {
    public static void main(String[] args) {
        MyThread myThread = new MyThread(); 	
        myThread.start();
        System.out.println(Thread.currentThread().getName() + " main()方法执行结束");
    }
}
```

运行结果

```java
main main()方法执行结束
Thread-0 run()方法正在执行...
```

## **实现 Runnable 接口**

1. 定义Runnable接口实现类MyRunnable，并重写run()方法
2. 创建MyRunnable实例myRunnable，以myRunnable作为target创建Thead对象，**该Thread对象才是真正的线程对象**
3. 调用线程对象的start()方法

```java
public class MyRunnable implements Runnable {
    @Override
    public void run() {
        System.out.println(Thread.currentThread().getName() + " run()方法执行中...");
    }
}
public class RunnableTest {
    public static void main(String[] args) {
        MyRunnable myRunnable = new MyRunnable();
        Thread thread = new Thread(myRunnable);
        thread.start();
        System.out.println(Thread.currentThread().getName() + " main()方法执行完成");
    }
}
```

执行结果

```java
main main()方法执行完成
Thread-0 run()方法执行中...
```

## **实现 Callable 接口**

1. 创建实现Callable接口的类myCallable
2. 以myCallable为参数创建FutureTask对象
3. 将FutureTask作为参数创建Thread对象
4. 调用线程对象的start()方法

```java
public class MyCallable implements Callable<Integer> {
    @Override
    public Integer call() {
        System.out.println(Thread.currentThread().getName() + " call()方法执行中...");
        return 1;
    }
}
public class CallableTest {
    public static void main(String[] args) {
        FutureTask<Integer> futureTask = new FutureTask<Integer>(new MyCallable());
        Thread thread = new Thread(futureTask);
        thread.start();

        try {
            Thread.sleep(1000);
            System.out.println("返回结果 " + futureTask.get());
        } catch (InterruptedException e) {
            e.printStackTrace();
        } catch (ExecutionException e) {
            e.printStackTrace();
        }
        System.out.println(Thread.currentThread().getName() + " main()方法执行完成");
    }
}
```

执行结果

```java
Thread-0 call()方法执行中...
返回结果 1
main main()方法执行完成
```

## **使用 Executors 工具类创建线程池**

Executors提供了一系列工厂方法用于创先线程池，返回的线程池都实现了ExecutorService接口。

主要有newFixedThreadPool，newCachedThreadPool，newSingleThreadExecutor，newScheduledThreadPool，后续详细介绍这四种线程池

```java
public class MyRunnable implements Runnable {
    @Override
    public void run() {
        System.out.println(Thread.currentThread().getName() + " run()方法执行中...");
    }
}

public class SingleThreadExecutorTest {
    public static void main(String[] args) {
        ExecutorService executorService = Executors.newSingleThreadExecutor();
        MyRunnable runnableTest = new MyRunnable();
        for (int i = 0; i < 5; i++) {
            executorService.execute(runnableTest);
        }
        System.out.println("线程任务开始执行");
        executorService.shutdown();
    }
}

```

执行结果

```java
线程任务开始执行
pool-1-thread-1 is running...
pool-1-thread-1 is running...
pool-1-thread-1 is running...
pool-1-thread-1 is running...
pool-1-thread-1 is running...

```

## runnable 和 callable 的区别

相同点

- 都是接口
- 都可以编写多线程程序
- 都采用Thread.start()启动线程

主要区别

- Runnable 接口 run 方法无返回值；Callable 接口 call 方法有返回值，是个泛型，和Future、FutureTask配合可以用来获取异步执行的结果
- Runnable 接口 run 方法只能抛出运行时异常，且无法捕获处理；Callable 接口 call 方法允许抛出异常，可以获取异常信息

**注**：Callalbe接口支持返回执行结果，需要调用FutureTask.get()得到，此方法会阻塞主进程的继续往下执行，如果不调用不会阻塞。

## 线程的 run()和 start()的区别

每个线程都是通过某个特定Thread对象所对应的方法run()来完成其操作的，run()方法称为线程体。通过调用Thread类的start()方法来启动一个线程。

start() 方法用于启动线程，run() 方法用于执行线程的运行时代码。run() 可以重复调用，而 start() 只能调用一次。

start()方法来启动一个线程，真正实现了多线程运行。调用start()方法无需等待run方法体代码执行完毕，可以直接继续执行其他的代码； 此时线程是处于就绪状态，并没有运行。 然后通过此Thread类调用方法run()来完成其运行状态， run()方法运行结束， 此线程终止。然后CPU再调度其它线程。

run()方法是在本线程里的，只是线程里的一个函数，而不是多线程的。 如果直接调用run()，其实就相当于是调用了一个普通函数而已，直接待用run()方法必须等待run()方法执行完毕才能执行下面的代码，所以执行路径还是只有一条，根本就没有线程的特征，所以在多线程执行时要使用start()方法而不是run()方法。

**为什么我们调用 start() 方法时会执行 run() 方法，为什么我们不能直接调用 run() 方法？**

new 一个 Thread，线程进入了新建状态。调用 start() 方法，会启动一个线程并使线程进入了就绪状态，当分配到时间片后就可以开始运行了。 start() 会执行线程的相应准备工作，然后自动执行 run() 方法的内容，这是真正的多线程工作。

而直接执行 run() 方法，会把 run 方法当成一个 main 线程下的普通方法去执行，并不会在某个线程中执行它，所以这并不是多线程工作。

总结： 调用 start 方法方可启动线程并使线程进入就绪状态，而 run 方法只是 thread 的一个普通方法调用，还是在主线程里执行。

## FutureTask

FutureTask 表示一个异步运算的任务。FutureTask 里面可以传入一个 Callable 的具体实现类，可以对这个异步运算的任务的结果进行等待获取、判断是否已经完成、取消任务等操作。只有当运算完成的时候结果才能取回，如果运算尚未完成 get 方法将会阻塞。一个 FutureTask 对象可以对调用了 Callable 和 Runnable 的对象进行包装，由于 FutureTask 也是Runnable 接口的实现类，所以 FutureTask 也可以放入线程池中。

# 为什么用线程池

1. 创建/销毁线程伴随着系统开销，过于频繁的创建/销毁线程，会很大程度上影响处理效率，应用能够更加充分合理地协调利用CPU、内存、网络、I/O等系统资源；
2. 线程并发数量过多，抢占系统资源从而导致阻塞，利用线程池管理并复用线程，控制最大并发数；
3. 使用线程池可以对线程进行一些简单的管理，实现任务线程队列缓存策略和拒绝机制，实现某些与时间相关的功能，例如定时执行、周期执行等，另外还可以隔离线程环境；


# ThreadPoolExecutor的重要参数
## corePoolSize：核心线程数
    * 核心线程会一直存活，即使没有任务需要执行
    * 当线程数小于核心线程数时，即使有线程空闲，线程池也会优先创建新线程处理
    * 设置allowCoreThreadTimeout=true（默认false）时，核心线程会超时关闭
  一般来说：
* CPU密集型：corePoolSize = CPU核数 + 1
* IO密集型：corePoolSize = CPU核数 * 2

## maximumPoolSize：最大线程数
    * 当线程数>=corePoolSize，且任务队列已满时。线程池会创建新线程来处理任务
    * 当线程数=maxPoolSize，且任务队列已满时，线程池会拒绝处理任务而抛出异常

## keepAliveTime：线程空闲时间
    * 当线程空闲时间达到keepAliveTime时，线程会退出，直到线程数量=corePoolSize
    * 如果allowCoreThreadTimeout=true，则会直到线程数量=0
## unit：线程空闲时间的单位
    * NANOSECONDS ： 1微毫秒 = 1微秒 / 1000
    * MICROSECONDS ： 1微秒 = 1毫秒 / 1000
    * MILLISECONDS ： 1毫秒 = 1秒 /1000
    * SECONDS ： 秒
    * MINUTES ： 分
    * HOURS ： 小时
    * DAYS ： 天

## workQueue：用于缓存任务的阻塞队列
常用的workQueue类型：
* SynchronousQueue：这个队列接收到任务的时候，会直接提交给线程处理，而不保留它，如果所有线程都在工作怎么办？那就新建一个线程来处理这个任务！所以为了保证不出现<线程数达到了maximumPoolSize而不能新建线程>的错误，使用这个类型队列的时候，maximumPoolSize一般指定成Integer.MAX_VALUE，即无限大
* LinkedBlockingQueue：这个队列接收到任务的时候，如果当前线程数小于核心线程数，则新建线程(核心线程)处理任务；如果当前线程数等于核心线程数，则进入队列等待。由于这个队列没有最大值限制，即所有超过核心线程数的任务都将被添加到队列中，这也就导致了maximumPoolSize的设定失效，因为总线程数永远不会超过corePoolSize
* ArrayBlockingQueue：可以限定队列的长度，接收到任务的时候，如果没有达到corePoolSize的值，则新建线程(核心线程)执行任务，如果达到了，则入队等候，如果队列已满，则新建线程(非核心线程)执行任务，又如果总线程数到了maximumPoolSize，并且队列也满了，则发生错误
* DelayQueue：队列内元素必须实现Delayed接口，这就意味着你传进去的任务必须先实现Delayed接口。这个队列接收到任务时，首先先入队，只有达到了指定的延时时间，才会执行任务

## handler：表示当workQueue已满，且池中的线程数达到maximumPoolSize时，线程池拒绝添加新任务时采取的策略。
    * AbortPolicy 丢弃任务，抛运行时异常 （默认）
    * CallerRunsPolicy 执行任务
    * DiscardPolicy 忽视，什么都不会发生
    * DiscardOldestPolicy 从队列中踢出最先进入队列（最后一个执行）的任务
    * 实现RejectedExecutionHandler接口，可自定义处理器

## threadFactory：指定创建线程的工厂
在阿里巴巴java开发手册以及一些sonar扫描规则中，需要手动创建线程池，和线程工厂，来对线程池进行命名。例如
```java
private static final ThreadFactory NAMED_THREAD_FACTORY = new ThreadFactoryBuilder()
            .setNameFormat("thread-name-%d").build();
    private static final ExecutorService THREAD_POOL = new ThreadPoolExecutor(1, 1,
            0L, TimeUnit.MILLISECONDS, new LinkedBlockingQueue<>(), NAMED_THREAD_FACTORY);
```
![线程池流程图](/blogImg/线程池流程图.png)

# 默认常用线程池

虽然是默认常用，但是如果加了sonar规则的话这几个还不能直接用。。还是要用线程池的构造函数自己去构造。
## newFixedThreadPool

创建固定线程数的线程池因为最大线程数和核心线程数相等，并且是无界队列，可控制线程最大并发数（同时执行的线程数），超出的线程会在队列中等待；

```java
public static ExecutorService newFixedThreadPool(int nThreads) {
        return new ThreadPoolExecutor(nThreads, nThreads,
                                      0L, TimeUnit.MILLISECONDS,
                                      new LinkedBlockingQueue<Runnable>());
    }
```
## newSingleThreadExecutor 

单任务队列的线程池,最大线程数和核心线程数都是1，无界队列，所有的任务都按照顺序进行执行；

```java
public static ExecutorService newSingleThreadExecutor() {
        return new FinalizableDelegatedExecutorService
            (new ThreadPoolExecutor(1, 1,
                                    0L, TimeUnit.MILLISECONDS,
                                    new LinkedBlockingQueue<Runnable>()));
    }
```
## ScheduledThreadPool

 支持定时周期性执行任务的线程池；

```java
 public static ScheduledExecutorService newScheduledThreadPool(int corePoolSize) {
        return new ScheduledThreadPoolExecutor(corePoolSize);
    }
```
## CachedThreadPool 

线程数无限制 有空闲线程则复用空闲线程，若无空闲线程则新建线程 一定程序减少频繁创建/销毁线程，减少系统开销.

```java
public static ExecutorService newCachedThreadPool() {
        return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                                      60L, TimeUnit.SECONDS,
                                      new SynchronousQueue<Runnable>());
    }
```
# 源码分析
## 任务的提交
submit方法源码
```java
public Future<?> submit(Runnable task) {
    if (task == null) throw new NullPointerException();
    RunnableFuture<Void> ftask = newTaskFor(task, null);
    execute(ftask);
    return ftask;
}
 
public <T> Future<T> submit(Callable<T> task) {
    if (task == null) throw new NullPointerException();
    RunnableFuture<T> ftask = newTaskFor(task);
    execute(ftask);
    return ftask;
}
public <T> Future<T> submit(Runnable task, T result) {
        if (task == null) throw new NullPointerException();
        RunnableFuture<T> ftask = newTaskFor(task, result);
        execute(ftask);
        return ftask;
}
```
&nbsp;&nbsp;&nbsp;&nbsp;这个方法一共有三个重载方法，分别是重写Callable和Runnable方法的重载。submit的实现方法位于抽象类AbstractExecutorService中，而此时execute方法还未实现（而是在AbstractExecutorService的继承类ThreadPoolExecutor中实现），可以看出无论哪个submit方法都最终调用了execute方法。
***
execute方法源码
```java
public void execute(Runnable command) {
    if (command == null)
        throw new NullPointerException();
 
    int c = ctl.get();
    if (workerCountOf(c) < corePoolSize) {
        if (addWorker(command, true))
            return;
        c = ctl.get();
    }
     
    if (isRunning(c) && workQueue.offer(command)) {
        int recheck = ctl.get();
        if (! isRunning(recheck) && remove(command))
            reject(command);
        else if (workerCountOf(recheck) == 0)
            addWorker(null, false);
    }
    else if (!addWorker(command, false))
        reject(command);
}
```
&nbsp;&nbsp;&nbsp;&nbsp;简要分析一下execute源码，执行一个Runnable对象时，首先通过workerCountOf(c)获取线程池中线程的数量，如果池中的数量小于corePoolSize就调用addWorker添加一个线程来执行这个任务。否则通过workQueue.offer(command)方法入列。如果入列成功还需要在一次判断池中的线程数，因为我们创建线程池时可能要求核心线程数量为0，所以我们必须使用addWorker(null, false)来创建一个临时线程去阻塞队列中获取任务来执行。

第二次addWorker是在工作线程为0时调用的，我们现在假设此时是创建核心线程，即`false`改为`true`；那么`addWorker`方法中`wc >= (core ? corePoolSize : maximumPoolSize)`这个地方会去判断当前工作线程是否大于核心线程，在高并发的情况下，会存在其他线程将工作线程的数量创建的大于核心线程数，导致返回`false`，并且不会创建新线程，虽然有工作线程的存在，但是会导致原本可以及时处理的任务，要去排队执行。

&nbsp;&nbsp;&nbsp;&nbsp;isRunning(c) 的作用是判断线程池是否处于运行状态，如果入列后发现线程池已经关闭，则出列。不需要在入列前判断线程池的状态，因为判断一个线程池工作处于RUNNING状态到执行入列操作这段时间，线程池可能被其它线程关闭了，所以提前判断毫无意义。
***
其中addWorker方法就是创建一个线程来执行Runnable对象
```java
private boolean addWorker(Runnable firstTask, boolean core) {
        retry:
        for (;;) {
            int c = ctl.get();
            int rs = runStateOf(c);

            // Check if queue empty only if necessary.
            //判断线程池的是否可以接收新任务
            if (rs >= SHUTDOWN &&
                ! (rs == SHUTDOWN &&
                   firstTask == null &&
                   ! workQueue.isEmpty()))
                return false;

            for (;;) {
            	//获取工作线程的数量
                int wc = workerCountOf(c);
                //判断工作线程的数量是否大于等于线程池的上限或者核心或者最大线程数
                if (wc >= CAPACITY ||
                    wc >= (core ? corePoolSize : maximumPoolSize))
                    return false;
                //使用cas增加工作线程数
                if (compareAndIncrementWorkerCount(c))
                    break retry;
                c = ctl.get();  // Re-read ctl
                //如果添加失败，并且线程池状态发生了改变，重来一遍
                if (runStateOf(c) != rs)
                    continue retry;
                // else CAS failed due to workerCount change; retry inner loop
            }
        }
		//上面的逻辑是考虑是否能够添加线程，如果可以使用cas来增加工作线程数量
		//下面正式启动线程
        boolean workerStarted = false;
        boolean workerAdded = false;
        Worker w = null;
        try {
        	//新建worker
            w = new Worker(firstTask);
            // 获取当前线程
            final Thread t = w.thread;
            if (t != null) {
            	//获取重入锁
                final ReentrantLock mainLock = this.mainLock;
                //锁住
                mainLock.lock();
                try {
                    // Recheck while holding lock.
                    // Back out on ThreadFactory failure or if
                    // shut down before lock acquired.
                    int rs = runStateOf(ctl.get());
					// rs < SHUTDOWN  -- 状态即为：RUNNING
					//rs == SHUTDOWN && firstTask == null 
                    if (rs < SHUTDOWN ||
                        (rs == SHUTDOWN && firstTask == null)) {
                        //如果线程已经启动，抛出异常
                        if (t.isAlive()) // precheck that t is startable
                            throw new IllegalThreadStateException();
                        //workers是一个HashSet，必须在锁住的情况下，操作
                        workers.add(w);
                        int s = workers.size();
                        //设置largestPoolSize ，标记workerAdded 
                        if (s > largestPoolSize)
                            largestPoolSize = s;
                        workerAdded = true;
                    }
                } finally {
                    mainLock.unlock();
                }
                //如果添加成功，启动线程
                if (workerAdded) {
                    t.start();
                    workerStarted = true;
                }
            }
        } finally {
        	//启动线程失败,回滚
            if (! workerStarted)
                addWorkerFailed(w);
        }
        return workerStarted;
    }

```
&nbsp;&nbsp;&nbsp;&nbsp;第一个参数firstTask不为null，则创建的线程就会先执行firstTask对象，然后去阻塞队列中取任务，否直接到阻塞队列中获取任务来执行。第二个参数，core参数为真，则用corePoolSize作为池中线程数量的最大值；为假，则以maximumPoolSize作为池中线程数量的最大值。
## 任务的执行
runWorker源码
```java
final void runWorker(Worker w) {
    Thread wt = Thread.currentThread();
    Runnable task = w.firstTask;
    w.firstTask = null;
    w.unlock(); // allow interrupts
    boolean completedAbruptly = true;
    try {
        while (task != null || (task = getTask()) != null) {
            w.lock();
            // If pool is stopping, ensure thread is interrupted;
            // if not, ensure thread is not interrupted.  This
            // requires a recheck in second case to deal with
            // shutdownNow race while clearing interrupt
            if ((runStateAtLeast(ctl.get(), STOP) ||
                 (Thread.interrupted() &&
                  runStateAtLeast(ctl.get(), STOP))) &&
                !wt.isInterrupted())
                wt.interrupt();
            try {
                beforeExecute(wt, task);
                Throwable thrown = null;
                try {
                    task.run();
                } catch (RuntimeException x) {
                    thrown = x; throw x;
                } catch (Error x) {
                    thrown = x; throw x;
                } catch (Throwable x) {
                    thrown = x; throw new Error(x);
                } finally {
                    afterExecute(task, thrown);
                }
            } finally {
                task = null;
                w.completedTasks++;
                w.unlock();
            }
        }
        completedAbruptly = false;
    } finally {
        processWorkerExit(w, completedAbruptly);
    }
}
```
 Thread的run方法实际上调用了Worker类的runWorker方法，而Worker类继承了AQS类，并实现了lock、unlock、trylock方法。但是这些方法不是真正意义上的锁，所以在代码中加锁操作和解锁操作没有成对出现。runWorker方法中获取到任务就“加锁”，完成任务后就“解锁”。也就是说在“加锁”到“解锁”的这段时间内，线程处于忙碌状态，而其它时间段，处于空闲状态。线程池就可以通过trylock方法来确定这个线程是否空闲。
***

 getTask方法的主要作用是从阻塞队列中获取任务。

       beforeExecute(wt, task)和afterExecute(task, thrown)是个钩子函数，如果我们需要在任务执行之前和任务执行以后进行一些操作，那么我们可以自定义一个继承ThreadPoolExecutor类，并覆盖这两个方法。
```java
private Runnable getTask() {
    boolean timedOut = false; // Did the last poll() time out?
 
    for (;;) {
        int c = ctl.get();
        int rs = runStateOf(c);
 
        // Check if queue empty only if necessary.
        if (rs >= SHUTDOWN && (rs >= STOP || workQueue.isEmpty())) {
            decrementWorkerCount();
            return null;
        }
 
        int wc = workerCountOf(c);
 
        // Are workers subject to culling?
        boolean timed = allowCoreThreadTimeOut || wc > corePoolSize;
 
        if ((wc > maximumPoolSize || (timed && timedOut))
            && (wc > 1 || workQueue.isEmpty())) {
            if (compareAndDecrementWorkerCount(c))
                return null;
            continue;
        }
 
        try {
            Runnable r = timed ?
                workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS) :
                workQueue.take();
            if (r != null)
                return r;
            timedOut = true;
        } catch (InterruptedException retry) {
            timedOut = false;
        }
    }
}
```
## 线程池的关闭
shutdown源码
```java
public void shutdown() {
    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    try {
        checkShutdownAccess();
        advanceRunState(SHUTDOWN);
        interruptIdleWorkers();
        onShutdown(); // hook for ScheduledThreadPoolExecutor
    } finally {
        mainLock.unlock();
    }
    tryTerminate();
}
```
advanceRunState(SHUTDOWN)的作用是通过CAS操作将线程池的状态更改为SHUTDOWN状态。
interruptIdleWorkers是对空闲的线程进行中断，它实际上调用了重载带参数的函数interruptIdleWorkers(false)
```java
private void interruptIdleWorkers(boolean onlyOne) {
    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    try {
        for (Worker w : workers) {
            Thread t = w.thread;
            if (!t.isInterrupted() && w.tryLock()) {
                try {
                    t.interrupt();
                } catch (SecurityException ignore) {
                } finally {
                    w.unlock();
                }
            }
            if (onlyOne)
                break;
        }
    } finally {
        mainLock.unlock();
    }
}
```
通过workers容器，遍历池中的线程，对每个线程进行tryLock()操作，如果成功说明线程空闲，则设置其中断标志位。而线程是否响应中断则由任务的编写者决定。

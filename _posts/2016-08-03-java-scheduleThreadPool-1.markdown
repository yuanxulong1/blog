---
layout: post
title:  "ScheduledExecutorService经常出现任务停止问题的排查"
date:   2016-08-02 09:00:49
categories: 问题解决
tags: Java
---

在线运行的一个后台同步数据任务，使用单线程定期从服务器端同步数据，最近经常性的出现任务停止的状态，java主干代码如下：

```java
public static void main(String[] args) {
	executorService.scheduleWithFixedDelay(new Runnable() {
		public void run() {
			//同步数据代码
		}
	}, 0, 5, TimeUnit.SECONDS)
}
```
使用jstack打印出任务停止后的线程堆栈信息如下：

```java
"XXXXXXXThread" daemon prio=10 tid=0x00007f9f1c5ff000 nid=0x47e5 waiting on condition [0x00007f9f7444e000]
   java.lang.Thread.State: WAITING (parking)
	at sun.misc.Unsafe.park(Native Method)
	- parking to wait for  <0x000000070812b3f0> (a java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionObject)
	at java.util.concurrent.locks.LockSupport.park(LockSupport.java:186)
	at java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionObject.await(AbstractQueuedSynchronizer.java:2043)
	at java.util.concurrent.ScheduledThreadPoolExecutor$DelayedWorkQueue.take(ScheduledThreadPoolExecutor.java:1079)
	at java.util.concurrent.ScheduledThreadPoolExecutor$DelayedWorkQueue.take(ScheduledThreadPoolExecutor.java:807)
	at java.util.concurrent.ThreadPoolExecutor.getTask(ThreadPoolExecutor.java:1068)
	at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1130)
	at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:615)
	at java.lang.Thread.run(Thread.java:745)

   Locked ownable synchronizers:
	- None
```
发现任务线程卡在这儿：

```java
at java.util.concurrent.ScheduledThreadPoolExecutor$DelayedWorkQueue.take(ScheduledThreadPoolExecutor.java:1079)
```
查看ScheduledThreadPoolExecutor的1079行代码如下：

```java
        public RunnableScheduledFuture take() throws InterruptedException {
            final ReentrantLock lock = this.lock;
            lock.lockInterruptibly();
            try {
                for (;;) {
                    RunnableScheduledFuture first = queue[0];
                    if (first == null)
                        available.await();  //1079行
            //后续代码省略       
```
也就是说，在从任务队列中取值时，任务队列为空，而且每次都卡在这儿，再看看是如何往任务队列中填充任务的，
提交task时：

```java
    public ScheduledFuture<?> scheduleWithFixedDelay(Runnable command,
                                                     long initialDelay,
                                                     long delay,
                                                     TimeUnit unit) {
        if (command == null || unit == null)
            throw new NullPointerException();
        if (delay <= 0)
            throw new IllegalArgumentException();
        ScheduledFutureTask<Void> sft =
            new ScheduledFutureTask<Void>(command,
                                          null,
                                          triggerTime(initialDelay, unit),
                                          unit.toNanos(-delay)); //该代码创建了定时执行的任务
        RunnableScheduledFuture<Void> t = decorateTask(command, sft);
        sft.outerTask = t;
        delayedExecute(t);//该处把任务放到队列中
        return t;
    }

```
delayedExecute方法，该方法仅仅是简单的把task放到任务队列中，然后检查处理线程池中是否可以创建线程（prestartCoreThread()）：

```java
    private void delayedExecute(RunnableScheduledFuture<?> task) {
        if (isShutdown())
            reject(task);
        else {
            super.getQueue().add(task);
           	if (isShutdown() &&
                !canRunInCurrentRunState(task.isPeriodic()) &&
                remove(task))
                task.cancel(false);
            else
                prestartCoreThread();//创建处理任务的线程池
        }
    }
```
prestartCoreThread()方法检查任务线程池中线程数，在需要的时候创建处理任务的线程并加入到线程池中，主要代码如下：

```java
    public boolean prestartCoreThread() {
        return workerCountOf(ctl.get()) < corePoolSize &&
            addWorker(null, true);
    }

    private boolean addWorker(Runnable firstTask, boolean core) {
    	//其它代码
    	...

    	Worker w = new Worker(firstTask);
        Thread t = w.thread;

        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
        	//其它代码   
        	...
            workers.add(w);
            ...  
            //其它代码
        } finally {
            mainLock.unlock();
        }
    }
```
处理任务的线程是Worker类，来看一下Worker类到底是什么（主要代码）：

```java
private final class Worker    extends AbstractQueuedSynchronizer
        implements Runnable{

    Worker(Runnable firstTask) {
        this.firstTask = firstTask;
        this.thread = getThreadFactory().newThread(this);
    }

    public void run() {
        runWorker(this);
    }   

    final void runWorker(Worker w) {
        Runnable task = w.firstTask;
        w.firstTask = null;
        boolean completedAbruptly = true;
        try {
            while (task != null || (task = getTask()) != null) {
                w.lock();
                clearInterruptsForTaskRun();
                try {
                    beforeExecute(w.thread, task);
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
}
```
从上面代码可以看到，worker仅仅是简单执行提交的任务，而我们提交的任务是ScheduledFutureTask类型，我们返回来再看一下ScheduledFutureTask的代码，我们提交的是一个定时任务，因此上面的逻辑可以忽略，：

```java
    private class ScheduledFutureTask<V>
            extends FutureTask<V> implements RunnableScheduledFuture<V> {

        public void run() {
            boolean periodic = isPeriodic();
            //代码不会执行到此处
            if (!canRunInCurrentRunState(periodic))
                cancel(false);
            else if (!periodic)
                ScheduledFutureTask.super.run();
            //代码在这儿执行
            else if (ScheduledFutureTask.super.runAndReset()) {
                setNextRunTime();
                reExecutePeriodic(outerTask);//这段代码把任务重新加入到队列中
            }
        }
    }
    //此处的runAndReset方法如下：
    protected boolean runAndReset() {
        return sync.innerRunAndReset();
    }

	boolean innerRunAndReset() {
        if (!compareAndSetState(READY, RUNNING))
            return false;
        try {
        	//这段代码执行如果抛出异常，则会返回false，  
        	//会导致ScheduledFutureTask的run方法不再往队列中添加任务，  
        	//因此队列为空，任务线程也就不再继续执行了
            runner = Thread.currentThread();
            if (getState() == RUNNING)
                callable.call(); // don't set result
            runner = null;
            return compareAndSetState(RUNNING, READY);
        } catch (Throwable ex) {
            setException(ex);
            return false;
        }
    }
```
从上面代码可以看出，如果在提交的任务代码执行中，如果抛出异常没有处理，会导致ScheduledFutureTask的run方法不再往队列中添加任务，因此任务队列为为空，这时候执行线程再从队列中取任务时，永远不可能取到任务了，因此出现了本文开头中出现的问题。



[jekyll]:      http://jekyllrb.com
[jekyll-gh]:   https://github.com/jekyll/jekyll
[jekyll-help]: https://github.com/jekyll/jekyll-help

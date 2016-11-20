---
date: 2016-11-20 22:16:12+08:00
layout: post
title: 一起学java并发编程（十五）：自定义并发类
thread: 17
categories: java
tags: java并发编程
---
Java并发类主要有两种，一种是管理并发状态的基础类，另一种就是创建并发任务的类。

### 同步基础类
<h4>状态管理</h4>
并发库中有许多存在状态依赖性的类，例如FutureTask、Semaphore和BlockingQueue等。这些类的一些操作中有着基于状态的前提条件。创建状态依赖类的最简单方法通常是在类库中现有状态依赖类的基础上进行构造。如果这些类库没有提供你需要的功能，那么还可以使用Java语言和类库提供的底层机制来构造自己的同步机制，包括内置的条件队列、显示的Condition对象以及AbstractQueuedSynchronizer框架。

依赖状态的操作可以一直阻塞直到可以继续执行，这比使它们先失败再实现起来要更为方便且更不易出错。可阻塞的状态依赖操作的加锁模式有些不同寻常，因为锁是在操作的执行过程中被释放与重新获取的。构成前提条件的状态变量必须由对象的锁来保护，从而使它们在测试前提条件的同时保持不变。如果前提条件尚未满足，就必须释放锁，以便其他线程可以修改对象的状态，否则，前提条件就永远无法变成真。在再次测试前提条件之前，必须重新获得锁。

在并发有界缓存（例如ArrayBlockingQueue）中对缓存的状态依赖管理，有几种实现方式。

第一种就是当状态未满足时，抛出一个异常或返回一个错误。

```java
public class GrumpyBoundedBuffer<V> extends BaseBoundedBuffer<V> {
	public GrumpyBoundedBuffer(int size) {
		super(size);
	}

	public synchronized void put(V v) throws BufferFullException {
		if (isFull()) {
			throw new BufferFullException();
		}
		doPut(v);
	}

	public synchronized V take() throws BufferEmptyException {
		if (isEmpty()) {
			throw new BufferEmptyException();
		}
		return doTake();
	}
}
```

尽管这种方式实现简单，但是使用起来麻烦，而且“缓存已满”或“缓存为空”并不是有界缓存的一个异常。调用者必须捕获异常做好重试准备。重试也有两种实现，一种是自旋等待，会浪费CPU资源，另一种是休眠一段时间再取，会导致低响应。

另一种方式是通过轮询保持阻塞直到对象进入正确的状态。

```java
public class SleepyBoundedBuffer<V> extends BaseBoundedBuffer<V> {
	public SleepyBoundedBuffer(int size) {
		super(size);
	}

	public void put(V v) throws InterruptedException {
		while (true) {
			synchronized (this) {
				if (!isFull()) {
					doPut(v);
					return;
				}
			}
			Thread.sleep(SLEEP_GRANULARITY);
		}
	}

	public V take() throws InterruptedException {
		while (true) {
			synchronized (this) {
				if (!isEmpty()) {
					return doTake();
				}
				Thread.sleep(SLEEP_GRANULARITY);
			}
		}
	}
}
```

这种实现方式比较复杂，但调用者使用简单。同时必须提供一种取消机制，这里通过中断来支持取消。这种方式同样需要在CPU资源和响应延迟之间进行权衡。

还有一种实现方式是使用条件队列。条件队列与传统队列的区别就是条件队列中的元素是一个个正在等待相关条件的线程。每个Java对象都可以作为一个锁，每个对象同样可以作为一个条件队列，并且Object中的wait、notify和notifyAll方法就构成了内部条件队列的API。Object.wait会自动释放锁，并请求操作系统挂起当前线程，从而使其他线程能够获得这个锁并修改对象的状态。当被挂起的线程醒来时，它将返回之前重新获取锁。

```java
public class BoundedBuffer<V> extends BaseBoundedBuffer<V> {
	public BoundedBuffer(int size) {
		super(size);
	}

	public synchronized void put(V v) throws InterruptedException {
		while (true) {
			wait();
		}
		doPut(v);
		notifyAll();
	}

	public synchronized V take() throws InterruptedException {
		while (isEmpty()) {
			wait();
		}
		V v = doTake();
		notifyAll();
		return v;
	}
}
```

条件队列在CPU效率、上下文切换和响应性进行了改进，功能上与通过“轮询与休眠”实现的效果是一样的。

这里为什么要使用synchronized同步操作的原因，是因为wait和notifyAll都需要获取对象的锁，而synchronized默认的锁就是对象实例。

当调用wait、notify或notifyAll等方法时，一定要持有与条件队列相关的锁。在检查条件谓语之后以及开始执行相应的操作之前，不要释放锁。

大多数情况下，notifyAll都是优先于notify的选择。只有同时满足下述条件后，才能用单一的notify取代notifyAll：
> 相同的等待者，只有一个条件谓词与条件队列相关，每个线程从wait返回后执行相同的逻辑；并且，一进一出。一个对条件变量的通知，至多激活一个线程执行。

<h4>Condition</h4>
内部条件队列有一些缺陷。每个内部锁只能有一个与之相关联的条件队列，例如，当调用notifyAll的时候，不仅会唤醒条件谓语为“队列非空”判断的线程，也会唤醒条件谓语为“队列已满”判断的线程。多个线程在同一个条件队列等待不同的条件谓语均被唤醒。这时就可以使用显式的Lock和Condition而不是内置锁和条件队列，这是一种更灵活的选择。

一个Condition和一个单独的Lock相关联，就像条件队列和单独的内部锁相关联一样：调用与Condition相关联的Lock的Lock.newCondition方法，可以创建一个Condition。Condition提供了比内置条件队列更丰富的功能：在每个锁上可存在多个等待、条件等待可以是可中断的或不可中断的、基于时限的等待，以及公平的或非公平的队列操作。

不同于内部条件队列，你可以让每个Lock都有任意数量的Condition对象。Condition对象继承了与之相关的锁的公平性特征：如果是公平的锁，线程会依照FIFO的顺序从Condition.await中被释放。

> 危险警告：wait、notify和notifyAll在Condition对象中的对等体是await、signal和signalAll。但是，Condition继承于Object，这意味着它也有wait和notify方法。一定要确保使用了正确的版本——await和signal！

```java
class ConditionBoundedBuffer<T> {
	protected final Lock lock = new ReentrantLock();
	private final Condition notFull = lock.newCondition();
	private final Condition notEmpty = lock.newCondition();
	private final T[] items = (T[]) new Object[BUFFER_SIZE];
	private int tail, head, count;

	public void put(T x) throws InterruptedException {
		lock.lock();
		try {
			while (count == items.length) {
				notFull.await();
			}
			items[tail] = x;
			if (++tail == items.length) {
				tail = 0;
			}
			++count;
			notEmpty.signal();
		} finally {
			lock.unlock();
		}
	}

	public T take() throws InterruptedException {
		lock.lock();
		try {
			while (count == items.length) {
				notEmpty.await();
			}
			T x = items[head];
			items[head] = null;
			if (++head == items.length) {
				head = 0;
			}
			--count;
			notFull.signal();
			return x;
		} finally {
			lock.unlock();
		}
	}
}
```

<h4>AbstractQueuedSynchronizer</h4>
在Java中有许多同步类，例如ReentrantLock和Semaphore两个接口，它们存在许多共同点，都可以控制一定数量的线程通过，都可以等待，还可以取消，都支持中断的、不可中断的以及限时操作。它们的实现其实都使用一个共同的基类AbstractQueuedSynchronizer（AQS）。

在基于AQS构建的同步器类中，最基本的操作包括各种形式的获取操作和释放操作。获取操作是一种依赖状态的操作，并且通常会阻塞。当执行释放操作时，所有在请求时被阻塞的线程都会开始执行。

AQS负责管理同步器类中的状态，它管理了一个整数状态信息，可以通过getState、setState以及compareAndSetState等protected类型方法来进行操作。
java.util.concurrent中的许多可阻塞类，例如ReentrantLock、Semaphore、ReetrantReadWriteLock、CountDownLatch、SynchronousQueue和FutureTask等都是基于AQS构建的。

### 定制并发管理类
我们更加常见的使用场景是定制并发管理类，包括线程的定制、生成等。

<h4>定制ThreadPoolExecutor类</h4>
基于ThreadPoolExecutor定制的类可以添加一些统计信息，例如任务的执行时长等。

```java
public class MyExecutor extends ThreadPoolExecutor {
	private ConcurrentHashMap<String, Date> startTimes;

	public MyExecutor(int corePoolSize, int maximumPoolSize, long keepAliveTime, TimeUnit unit, BlockingQueue<Runnable> workQueue) {
		super(corePoolSize, maximumPoolSize, keepAliveTime, unit, workQueue);
		startTimes = new ConcurrentHashMap<String, Date>();
	}

	@Override
	public void shutdown() {
		System.out.println("MyExecutor: goging to shutdown.");
		System.out.println("MyExecutor: Executed tasks:" + getCompletedTaskCount());
		System.out.println("MyExecutor: Running tasks:" + getActiveCount());
		System.out.println("MyExecutor: Pending tasks:" + getQueue().size());
		super.shutdown();
	}

	@Override
	protected void beforeExecute(Thread t, Runnable r) {
		System.out.println("MyExecutor: A task is beginning:" + t.getName() + ":" + r.hashCode());
		startTimes.put(String.valueOf(r.hashCode()), new Date());
	}

	@Override
	protected void afterExecute(Runnable r, Throwable t) {
		Future<?> result = (Future<?>) r;
		try {
			System.out.println("*******************************");
			System.out.println("MyExecutor: A task is finishing.");
			System.out.println("MyExecutor: Result:" + result.get());
			Date startDate = startTimes.remove(String.valueOf(r.hashCode()));
			Date finishDate = new Date();
			long diff = finishDate.getTime() - startDate.getTime();
			System.out.println("MyExecutor: Duration: " + diff);
			System.out.println("*********************************");
		} catch (Exception e) {
			e.printStackTrace();
		}
		super.afterExecute(r, t);
	}
}
```

<h4>在Executor对象中使用ThreadFactory</h4>
可以继承现有的ThreadFactory类，实现一个统计线程数和指定线程名的ThreadFactory。

```java
public class MyThreadFactory implements ThreadFactory {
	private int counter;
	private String prefix;

	public MyThreadFactory(String prefix) {
		this.prefix = prefix;
		counter = 1;
	}

	@Override
	public Thread newThread(Runnable r) {
		MyThread myThread = new MyThread(r, prefix + "-" + counter);
		counter++;
		return myThread;
	}
}
```
<h4>定制运行在定时线程池中的任务</h4>
定时线程池可以执行两类特殊任务：延迟任务（Delayed Task）和周期性任务（Periodic Task）。可以实现RunnableScheduledFuture接口来执行延迟任务和周期性任务。

创建一个定制的MyScheduledTask类，覆写父类的一些方法，定制自己的行为。

```java
public class MyScheduledTask<V> extends FutureTask<V> implements RunnableScheduledFuture<V> {
	private RunnableScheduledFuture<V> task;
	private ScheduledThreadPoolExecutor executor;
	private long period;
	private long startDate;

	public MyScheduledTask(Runnable runnable, V result, RunnableScheduledFuture<V> task, ScheduledThreadPoolExecutor executor) {
		super(runnable, result);
		this.task = task;
		this.executor = executor;
	}

	@Override
	public long getDelay(TimeUnit unit) {
		if (!isPeriodic()) {
			return task.getDelay(unit);
		} else {
			if (startDate == 0) {
				return task.getDelay(unit);
			} else {
				Date now = new Date();
				long delay = startDate - now.getTime();
				return unit.convert(delay, TimeUnit.MILLISECONDS);
			}
		}
	}

	@Override
	public int compareTo(Delayed o) {
		return task.compareTo(o);
	}

	@Override
	public boolean isPeriodic() {
		return task.isPeriodic();
	}

	@Override
	public void run() {
		if (isPeriodic() && !executor.isShutdown()) {
			Date now = new Date();
			startDate = now.getTime() + period;
			executor.getQueue().add(this);
		}
		System.out.println("Pre-MyScheduledTask: " + new Date());
		System.out.println("MyScheduledTask: is periodic:" + isPeriodic());
		super.runAndReset();
		System.out.println("Post-MyScheduledTask: " + new Date());
	}

	public void setPeriod(long period) {
		this.period = period;
	}
}
```

创建一个ScheduledThreadPoolExecutor类，用来执行MyScheduledTask。

```java
public class MyScheduledThreadPoolExecutor extends ScheduledThreadPoolExecutor {

	public MyScheduledThreadPoolExecutor(int corePoolSize) {
		super(corePoolSize);
	}

	@Override
	protected <V> RunnableScheduledFuture<V> decorateTask(Runnable runnable, RunnableScheduledFuture<V> task) {
		MyScheduledTask<V> myTask = new MyScheduledTask<V>(runnable, null, task, this);
		return myTask;
	}

	@Override
	public ScheduledFuture<?> scheduleAtFixedRate(Runnable command, long initialDelay, long period, TimeUnit unit) {
		ScheduledFuture<?> task = super.scheduleAtFixedRate(command, initialDelay, period, unit);
		MyScheduledTask<?> myTask = (MyScheduledTask<?>) task;
		myTask.setPeriod(TimeUnit.MILLISECONDS.convert(period, unit));
		return myTask;
	}
}
```

创建一个线程类，使线程休眠2秒。

```java
public class Task implements Runnable {

	@Override
	public void run() {
		System.out.println("task: begin.");
		try {
			TimeUnit.SECONDS.sleep(2);
		} catch (InterruptedException e) {
			e.printStackTrace();
		}
		System.out.println("task: end.");
	}
}
```

执行定制的线程。

```java
public class Main {
	public static void main(String[] args) throws InterruptedException {
		MyScheduledThreadPoolExecutor executor = new MyScheduledThreadPoolExecutor(2);
		Task task = new Task();
		System.out.println("main: " + new Date());
		TimeUnit.SECONDS.sleep(2);
		task = new Task();
		System.out.println("main: " + new Date());
		executor.scheduleAtFixedRate(task, 1, 3, TimeUnit.SECONDS);
		TimeUnit.SECONDS.sleep(10);
		executor.shutdown();
		executor.awaitTermination(1, TimeUnit.DAYS);
		System.out.println("main: end of the propram.");
	}
}
```

<h4>实现定制Lock类</h4>
继承AQS，实现一个自己定制的AQS类。

```java
public class MyAbstractQueuedSynchronizer extends AbstractQueuedSynchronizer {

	private static final long serialVersionUID = 1L;
	private AtomicInteger state;

	public MyAbstractQueuedSynchronizer() {
		state = new AtomicInteger(0);
	}

	@Override
	protected boolean tryAcquire(int arg) {
		return state.compareAndSet(0, 1);
	}

	@Override
	protected boolean tryRelease(int arg) {
		return state.compareAndSet(1, 0);
	}

}
```

使用AQS创建一个自定义的锁。

```java
public class MyLock implements Lock {
	private AbstractQueuedSynchronizer sync;

	public MyLock() {
		sync = new MyAbstractQueuedSynchronizer();
	}

	@Override
	public void lock() {
		sync.acquire(1);
	}

	@Override
	public void lockInterruptibly() throws InterruptedException {
		sync.acquireSharedInterruptibly(1);
	}

	@Override
	public boolean tryLock() {
		try {
			return sync.tryAcquireNanos(1, 1000);
		} catch (InterruptedException e) {
			e.printStackTrace();
			return false;
		}
	}

	@Override
	public boolean tryLock(long time, TimeUnit unit) throws InterruptedException {
		return sync.tryAcquireNanos(1, TimeUnit.NANOSECONDS.convert(time, unit));
	}

	@Override
	public void unlock() {
		sync.release(1);
	}

	@Override
	public Condition newCondition() {
		return sync.new ConditionObject();
	}

}
```

使用定制锁实现线程，线程中使用自己定制的锁实现同步。

```java
public class Task implements Runnable {
	private MyLock lock;
	private String name;

	public Task(MyLock lock, String name) {
		this.lock = lock;
		this.name = name;
	}

	@Override
	public void run() {
		lock.lock();
		System.out.println("task: " + name + ": take the lock.");
		try {
			TimeUnit.SECONDS.sleep(2);
			System.out.println("task:" + name + " free the lock");
		} catch (InterruptedException e) {
			e.printStackTrace();
		} finally {
			lock.unlock();
		}

	}

}
```

执行上面的线程。

```java
public class Main {
	public static void main(String[] args) {
		MyLock lock = new MyLock();
		for (int i = 0; i < 10; i++) {
			Task task = new Task(lock, "task-" + i);
			Thread thread = new Thread(task);
			thread.start();
		}
		boolean value = false;
		try {
			do {
				value = lock.tryLock(1, TimeUnit.SECONDS);
				if (!value) {
					System.out.println("main: trying to get the lock.");
				}
			} while (!value);
		} catch (InterruptedException e) {
			e.printStackTrace();
		} finally {
			System.out.println("main: got the lock");
			lock.unlock();
		}
		System.out.println("main: end of the program");
	}
}
```

<h4>实现基于优先级的传输队列</h4>
LinkedTransferQueue中的元素按到达的先后顺序进行存储，所以早到的被优先消费。在PriorityBlockingQueue中，元素按顺序存储。这些元素必须实现Comparable接口，并实现compareTo()方法。我们可以基于这两个类实现一种按照指定优先级被消费的队列。

```java
public class MyPriorityTransferQueue<E> extends PriorityBlockingQueue<E> implements TransferQueue<E> {

	private static final long serialVersionUID = 1L;
	private AtomicInteger counter;
	private LinkedBlockingQueue<E> transfered;
	private ReentrantLock lock;

	public MyPriorityTransferQueue() {
		this.counter = new AtomicInteger(0);
		this.transfered = new LinkedBlockingQueue<E>();
		this.lock = new ReentrantLock();
	}

	@Override
	public boolean tryTransfer(E e) {
		lock.lock();
		boolean value;
		if (counter.get() == 0) {
			value = false;
		} else {
			put(e);
			value = true;
		}
		lock.unlock();
		return value;
	}

	@Override
	public void transfer(E e) throws InterruptedException {
		lock.lock();
		if (counter.get() != 0) {
			put(e);
			lock.unlock();
		} else {
			transfered.add(e);
			lock.unlock();
			synchronized (e) {
				e.wait();
			}
		}
	}

	@Override
	public boolean tryTransfer(E e, long timeout, TimeUnit unit) throws InterruptedException {
		lock.lock();
		if (counter.get() != 0) {
			put(e);
			lock.unlock();
			return true;
		} else {
			transfered.add(e);
			long newTimeout = TimeUnit.MILLISECONDS.convert(timeout, unit);
			lock.unlock();
			e.wait(newTimeout);
			lock.lock();
			if (transfered.contains(e)) {
				transfered.remove(e);
				lock.unlock();
				return false;
			} else {
				lock.unlock();
				return true;
			}
		}
	}

	@Override
	public boolean hasWaitingConsumer() {
		return (counter.get() != 0);
	}

	@Override
	public int getWaitingConsumerCount() {
		return counter.get();
	}

	@Override
	public E take() throws InterruptedException {
		lock.lock();
		counter.incrementAndGet();
		E value = transfered.poll();
		if (value == null) {
			lock.unlock();
			value = super.take();
			lock.lock();
		} else {
			synchronized (value) {
				value.notify();
			}
		}
		counter.decrementAndGet();
		lock.unlock();
		return value;
	}
}
```

实现Event类，并实现优先级的排序方法。

```java
public class Event implements Comparable<Event> {
	private String thread;
	private int priority;

	public Event(String thread, int priority) {
		this.thread = thread;
		this.priority = priority;
	}

	public String getThread() {
		return thread;
	}

	public int getPriority() {
		return priority;
	}

	@Override
	public int compareTo(Event o) {
		if (this.priority > o.getPriority()) {
			return -1;
		} else if (this.priority < o.getPriority()) {
			return 1;
		}
		return 0;
	}

}
```

Producer类向队列生产数据。

```java
public class Producer implements Runnable {
	private MyPriorityTransferQueue<Event> buffer;

	public Producer(MyPriorityTransferQueue<Event> buffer) {
		this.buffer = buffer;
	}

	@Override
	public void run() {
		for (int i = 0; i < 100; i++) {
			Event event = new Event(Thread.currentThread().getName(), i);
			buffer.put(event);
		}
	}

}
```

Consumer类从队列中消费数据。

```java
public class Consumer implements Runnable {
	private MyPriorityTransferQueue<Event> buffer;

	public Consumer(MyPriorityTransferQueue<Event> buffer) {
		this.buffer = buffer;
	}

	@Override
	public void run() {
		for (int i = 0; i < 1002; i++) {
			try {
				Event value = buffer.take();
				System.out.println("Consumer: " + value.getThread() + ":" + value.getPriority());
			} catch (InterruptedException e) {
				e.printStackTrace();
			}
		}
	}
}
```

执行并查看输出结果。

```java
public class Main {
	public static void main(String[] args) throws InterruptedException {
		MyPriorityTransferQueue<Event> buffer = new MyPriorityTransferQueue<Event>();
		Producer producer = new Producer(buffer);
		Thread[] producerThreads = new Thread[10];
		for (int i = 0; i < producerThreads.length; i++) {
			producerThreads[i] = new Thread(producer);
			producerThreads[i].start();
		}
		Consumer consumer = new Consumer(buffer);
		Thread consumerThread = new Thread(consumer);
		consumerThread.start();
		System.out.println("main: buffer: consumer count:" + buffer.getWaitingConsumerCount());
		Event myEvent = new Event("core event", 0);
		buffer.transfer(myEvent);
		System.out.println("main: my event has been transfered.");
		for (int i = 0; i < producerThreads.length; i++) {
			producerThreads[i].join();
		}
		TimeUnit.SECONDS.sleep(1);
		System.out.println("main: buffer: consumer count:" + buffer.getWaitingConsumerCount());
		myEvent = new Event("core event 2", 0);
		buffer.transfer(myEvent);
		consumerThread.join();
		System.out.println("main: end of the program");
	}
}
```

多线程的基本知识基本上就是这些。如果想深入了解多线程的底层实现，需要设计JVM和操作系统的底层实现了。
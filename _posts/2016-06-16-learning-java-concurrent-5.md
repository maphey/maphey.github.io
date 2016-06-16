---
date: 2016-06-16 21:16:12+08:00
layout: post
title: 一起学java并发编程（五）：基础构建模块
thread: 7
categories: java
tags: java并发编程
---
### 同步容器的迭代问题
<p>好的同步容器遵守一个支持客户端加锁的同步策略，因此只要我们知道应该使用哪一个锁，就有可以针对其他的容器操作创建新的原子操作。同步容器类通过对它的对象自身进行加锁，保护它的每一个方法。</p>
<p>例如，同步容器Vector有两个复合操作：</p>
```java
public static <T> T getLast(Vector<T> list) {
	int lastIndex = list.size() - 1;
	return list.get(lastIndex);
}

public static <T> void deleteLast(Vector<T> list) {
	int lastIndex = list.size() - 1;
	list.remove(lastIndex);
}
```
<p>如果有两个个线程同时调用getLast和deleteLast，可能会出错，因为这两个复合操作没有同步成原子操作。为了将这两个复合操作同步成原子操作，我们需要获取容器的锁，确保Vector在获取size和修改size之间不会发生变化。</p>
```java
public static <T> T getLast(Vector<T> list) {
	synchronized (list) {
		int lastIndex = list.size() - 1;
		return list.get(lastIndex);
	}
}

public static <T> void deleteLast(Vector<T> list) {
	synchronized (list) {
		int lastIndex = list.size() - 1;
		list.remove(lastIndex);
	}
}
```
<p>这种情况如果出现在迭代中会更加的麻烦。</p>
```java
synchronized (vector) {
	for (int i = 0; i< vector.size(); i++) {
		doSomething(vector.get(i));
	}
}
```
<p>如果上面的代码不同步，就不是线程安全的。如果向上面这样同步，就完全阻止了其他线程的并发访问，消弱了并发性。</p>
<p>在迭代访问中，如果处理的容器很大，或者处理的任务耗时较长，就会严重影响性能。java内置的迭代器（iterator）没有在迭代期间对容器进行加锁，而是采用一种“及时失败”的策略，即如果容器在迭代期间被修改，就会抛出ConcurrentModificationException。在迭代期间，对容器加锁的一个替代方法是复制容器。因为复制是线程限制的，没有其他线程能够在迭代期间对其进行修改。</p>
<p>还有一种隐形迭代器，调用容器的toString，hashCode或者equals方法可能会迭代其中的每个元素。</p>
<p>下面的代码看起来是线程安全的，但是set容器不是线程安全的，addTenThings也没有被同步起来，所以会抛出ConcurrentModificationException异常。</p>
```java
import java.util.HashSet;
import java.util.Random;
import java.util.Set;

public class HiddenIterator {
	private final Set<Integer> set = new HashSet<Integer>();

	public synchronized void add(Integer i) {
		set.add(i);
	}

	public synchronized void remove(Integer i) {
		set.remove(i);
	}

	public void addTenThings() {
		Random r = new Random();
		for (int i = 0; i < 10; i++) {
			add(r.nextInt());
			System.out.println("debug: added ten elements to " + set);
		}
	}
}
```
<p>如果将HashSet包装为synchronizedSet，封装了同步，就是线程安全的了。</p>

### 并发容器
<p>同步容器使用串行访问实现线程安全，但是降低了多线程访问并发性。并发容器就是为多线程设计的。用并发容器替换同步容器，这种方法以有很小风险带来了可扩展性显著的提高。</p>
<p>Queue和BlockingQueue就是为了提供更加高效的并发实现而新增的。前者是非阻塞的，后者提供了阻塞的插入和获取操作。</p>
<p>ConcurrentSkipListMap是SortedMap的并发实现；ConcurrentSkipListSet是SortedSet的并发实现；ConcurrentHashMap是HashMap的并发实现。</p>
<p>ConcurrentHashMap采用了更加细化的锁机制，分离锁的机制。这个机制允许更深层次的共享访问。ConcurrentHashMap不能在独占访问中被加锁，我们不能在客户端使用加锁机制来创建新的原子操作。</p>
<p>“写入时赋值（copy-on-write）”容器基于不可变对象被正确发布，就不需要使用特殊的同步机制进行同步。每次修改时，它们就会重新创建并发布一个新的容器拷贝，以此实现可变性。CopyOnWriteArrayList就是同步List的一个并发替代品。容器迭代器会保留一个底层基础数组的引用。这个数组作为迭代器的起点，永远不会被修改。显然，这只适合适用于迭代频率高于修改频率的情况。</p>

### 生产者和消费者
<p>在构建高可靠的应用程序时，有界队列是一种强大的资源管理工具：它们能抑制并防止产生过多的工作项，使应用程序在负荷过载的情况下变得更加健壮。</p>
<p>阻塞队列提供了可阻塞的take和put方法，它们与可定时的offer和poll是等价的（offer和poll如果不能生效就会返回失败，供后续处理）。如果Queue已经满了，put方法会被阻塞直到有空间可以；如果Queue为空，take会被阻塞直到Queue有元素可用。阻塞队列天生支持生产者-消费者模式。</p>
<p>生产者-消费者模式解耦了生产者类和消费者类之间的相互依赖。两者可以以不同的速度生产或消费数据，并发执行。简化了工作负荷的管理。</p>
<p>java类库提供了LinkedBlockingQueue和ArrayBlockingQueue实现LinkedList和ArrayList的并发版本。PriorityBlockingQueue是一个按照优先级排序的队列。这些并发容器具体使用将在之后的博客中介绍。还有一个不维护队列，直接将生产者产生的任务直接交给消费者的BlockingQueue，SynchronousQueue。它不维护队列，只维护排队的线程，减少了生产者和消费者之间移动数据的延迟时间。</p>
<p>假设我们有一个创建文件索引的任务，我们将搜索文件与创建文件索引使用生产者-消费者模式实现。</p>
```java
import java.io.File;
import java.io.FileFilter;
import java.util.concurrent.BlockingQueue;

public class FileCrawler implements Runnable {
	private final BlockingQueue<File> fileQueue;
	private final FileFilter fileFilter;
	private final File root;

	public FileCrawler(BlockingQueue<File> fileQueue, FileFilter fileFilter, File root) {
		this.fileQueue = fileQueue;
		this.fileFilter = fileFilter;
		this.root = root;
	}

	@Override
	public void run() {
		try {
			crawl(root);
		} catch (InterruptedException e) {
			e.printStackTrace();
		}
	}

	private void crawl(File root) throws InterruptedException {
		File[] entries = root.listFiles(fileFilter);
		if (entries != null) {
			for (File entry : entries) {
				if (entry.isDirectory()) {
					crawl(entry);
				} else if (!fileQueue.contains(entry)) {
					fileQueue.put(entry);
				}
			}
		}
	}
}
```
```java
import java.io.File;
import java.util.concurrent.BlockingQueue;

public class Indexer implements Runnable {
	private final BlockingQueue<File> queue;

	public Indexer(BlockingQueue<File> queue) {
		this.queue = queue;
	}

	@Override
	public void run() {
		while (true) {
			try {
				indexFile(queue.take());
			} catch (InterruptedException e) {
				e.printStackTrace();
			}
		}
	}

	private void indexFile(File take) {
		// create index...
	}
}
```
<p>分别创建生产者和消费者线程，生产者向队列加入任务，消费者消费任务：</p>
```java
import java.io.File;
import java.io.FileFilter;
import java.util.concurrent.BlockingQueue;
import java.util.concurrent.LinkedBlockingDeque;

public class Main {
	public static void startIndexing(File[] roots) {
		BlockingQueue<File> queue = new LinkedBlockingDeque<File>(BOUND);
		FileFilter filter = new FileFilter() {
			@Override
			public boolean accept(File pathname) {
				return true;
			}
		};
		for (File root : roots) {
			new Thread(new FileCrawler(queue, filter, root)).start();
		}
		for (int i = 0; i < N_CONSUMERS; i++) {
			new Thread(new Indexer(queue)).start();
		}
	}
}
```
<p>除了单向队列，java还有两个双端队列容器Deque和BlockingDeque。可以用来实现窃取工作模式。每个消费者消费完自己队列的任务后，可以从其他队列的末尾取任务。因为消费者线程不会竞争同一个共享任务队列，所以窃取工作模式比传统的生产者-消费者模式更具可伸缩性。</p>

### 阻塞和可中断的方法
<p>Thread提供了interrupt方法，用来中断一个线程，或者查询某个线程是否已经被中断。</p>
<p>当你的代码调用了一个会抛出InterruptException的方法时，这个方法就成了一个阻塞方法，要为响应中断做好准备。</p>
<p>有两种方法处理中断：一是传递InterruptException。可以直接向上传递这个异常，或者先捕获，进行一些资源清理，再向上抛出这个异常。二是恢复中断，在当前线程中调用interrupt恢复中断。如下所示：</p>
```java
public class TaskRunnable implements Runnable {
	BlockingQueue<Task> queue;

	@Override
	public void run() {
		try {
			processTask(queue.take());
		} catch (InterruptedException e) {
			// 恢复中断状态
			Thread.currentThread().interrupt();
		}
	}

	private void processTask(Task take) {
		// process task...
	}
}
```
<p>但是不要掩盖这个异常，捕获了却不做任务处理。只有一种情况允许掩盖异常：扩展Thread，并因此控制了所有处于调用栈上层的代码。</p>

### Synchronizer
<p>Synchronizer是一个对象，它根据本身的状态调节线程的控制流。阻塞队列可以扮演一个Synchronizer的角色：其他类型的Synchronizer包括信号量（semaphore）、关卡（barrier）以及闭锁（latch）。也可以创建自己的Synchronizer。</p>
<p>闭锁（latch）可以延迟线程的进度直到线程到达终止状态。闭锁可以用来确保特定活动直到其他活动完成后才发生。闭锁一旦到达终点状态，它就再也不能改变状态了，永远保持敞开。</p>
<p>CountDownLatch是一个灵活的闭锁实现。允许一个或多个线程等待一个事件集的发生。闭锁的状态包括一个计数器，初始化为一个正数，用来表示需要等待的事件数。countDown表示对计数器做减操作，await等待计数器达到零。</p>
```java
import java.util.concurrent.CountDownLatch;

public class TestHarness {
	public long timeTasks(int nThreads, final Runnable task) throws InterruptedException {
		final CountDownLatch startGate = new CountDownLatch(1);
		final CountDownLatch endGate = new CountDownLatch(nThreads);
		for (int i = 0; i < nThreads; i++) {
			Thread t = new Thread() {
				@Override
				public void run() {
					try {
						startGate.await();
						task.run();
					} catch (InterruptedException e) {
						e.printStackTrace();
					} finally {
						endGate.countDown();
					}
				}
			};
			t.start();
		}
		long start = System.nanoTime();
		startGate.countDown();
		endGate.await();
		long end = System.nanoTime();
		return end - start;
	}
}
```
<p>这个例子有一个开始闭锁，一个结束闭锁。每个工作线程等待开始闭锁打开，主线程等待所有结束闭锁打开。</p>
<p>FutureTask同样可以作为闭锁。FutureTask的实现描述了一个抽象的可携带结果的计算。FutureTask的计算是通过Callable实现的，它等价于一个可携带结果的Runnable。</p>
```java
import java.util.concurrent.Callable;
import java.util.concurrent.ExecutionException;
import java.util.concurrent.FutureTask;

public class PreLoader {
	private final FutureTask<ProductInfo> future = new FutureTask<ProductInfo>(new Callable<ProductInfo>() {
		@Override
		public ProductInfo call() throws Exception {
			return loadProductInfo();
		}
	});
	private final Thread thread = new Thread(future);

	public void start() {
		thread.start();
	}

	public ProductInfo get() throws InterruptedException, ExecutionException {
		return future.get();
	}
}
```
<p>信号量用来控制能够同时访问某特定资源的活动的数量，或者同时执行某一给定操作的数量。计数信号量可以用来实现资源池或者给一个容器限定边界。</p>
```java
import java.util.Collections;
import java.util.HashSet;
import java.util.Set;
import java.util.concurrent.Semaphore;

public class BoundedHashSet<T> {
	private final Set<T> set;
	private final Semaphore sem;

	public BoundedHashSet(int bound) {
		this.set = Collections.synchronizedSet(new HashSet<T>());
		sem = new Semaphore(bound);
	}

	public boolean add(T o) throws InterruptedException {
		sem.acquire();
		boolean wasAdd = false;
		try {
			wasAdd = set.add(o);
			return wasAdd;
		} finally {
			if (!wasAdd) {
				sem.release();
			}
		}
	}

	public boolean remove(Object o) {
		boolean wasRemoved = set.remove(o);
		if (wasRemoved) {
			sem.release();
		}
		return wasRemoved;
	}
}
```
<p>关卡类似于闭锁，它们都能够阻塞一组线程，直到某些事件发生。其中关卡与闭锁关键的不同在于，所有线程必须同时到达关卡点，才能继续处理。闭锁等待的是事件，关卡等待的是其他线程。</p>
<p>关卡在成功打开后，会重置，以备下一次使用。当有线程到达关卡等待其他线程到达以达到关卡的开放条件时，该线程会被置于阻塞状态。如果关卡超时或者阻塞的线程被中断，那么关卡就是失败的，所有对await未完成的调用都通过BrokenBarrierException终止。如果成功通过关卡，await为每一个线程返回一个唯一的到达索引号，可以用它来选举产生一个领导，在下一次迭代中承担一些特殊工作。</p>
```java
import java.util.concurrent.BrokenBarrierException;
import java.util.concurrent.CyclicBarrier;

public class CellularAutomata {
	private final Board mainBoard;
	private final CyclicBarrier barrier;
	private final Worker[] workers;

	public CellularAutomata(Board Board) {
		this.mainBoard = Board;
		int count = Runtime.getRuntime().availableProcessors();
		barrier = new CyclicBarrier(count, new Runnable() {

			@Override
			public void run() {
				mainBoard.commitNewValues();
			}
		});
		this.workers = new Worker[count];
		for (int i = 0; i < count; i++) {
			workers[i] = new Worker(mainBoard.getSubBoard(count, i));
		}
	}

	public void start() {
		for (int i = 0; i < workers.length; i++) {
			new Thread(workers[i]).start();
		}
		mainBoard.waitForConvergence();
	}

	private class Worker implements Runnable {
		private final Board board;

		public Worker(Board board) {
			this.board = board;
		}

		@Override
		public void run() {
			while (!board.hasConverged()) {
				for (int x = 0; x < board.getMaxX(); x++) {
					for (int y = 0; y < board.getMaxY(); y++) {
						board.setNewValue(x, y, computeValue(x, y));
					}
				}
				try {
					barrier.await();
				} catch (InterruptedException | BrokenBarrierException e) {
					return;
				}
			}
		}
	}
}
```
<p>CellularAutomata是一个计算细胞的自动化模拟，让所有的线程完成子任务完成后，再合并结果。</p>
<p>Exchanger是关卡的另一种形式，它是一种两步关卡。</p>

### 建立高效的多线程缓存
<p>我们可以使用一个Map来模拟缓存，为了保证线程安全性，我们需要使用锁保证线程同步。</p>
```java
public interface Computable<A, V> {
	V compute(A arg) throws InterruptedException;
}
```
```java
public class ExpensiveFunction implements Computable<String, Integer> {

	@Override
	public Integer compute(String arg) throws InterruptedException {
		return Integer.valueOf(arg);
	}

}
```
```java
import java.util.HashMap;
import java.util.Map;

public class Memoizer1<A, V> implements Computable<A, V> {
	private final Map<A, V> cache = new HashMap<A, V>();
	private final Computable<A, V> c;

	public Memoizer1(Computable<A, V> c) {
		this.c = c;
	}

	@Override
	public synchronized V compute(A arg) throws InterruptedException {
		V result = cache.get(arg);
		if (result == null) {
			result = c.compute(arg);
			cache.put(arg, result);
		}
		return result;
	}
}
```
<p>Memoizer1使用了一种保守的方法，对整个compute进行加锁。尽管确保了线程安全性，但是每次只能有一个线程进行计算，影响了可伸缩性。如果计算时间很长甚至比不使用缓存耗时更长。</p>
```java
import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;

public class Memozer2<A, V> implements Computable<A, V> {
	private final Map<A, V> cache = new ConcurrentHashMap<A, V>();
	private final Computable<A, V> c;

	public Memozer2(Computable<A, V> c) {
		this.c = c;
	}

	@Override
	public V compute(A arg) throws InterruptedException {
		V result = cache.get(arg);
		if (result == null) {
			result = c.compute(arg);
			cache.put(arg, result);
		}
		return result;
	}
}
```
<p>在Memozer2中，我们使用线程安全的ConcurrentHashMap代替了锁。这种方式提高了并发性，尽管缓存容器本身是线程安全的，但是由于compute操作不是原子的，可能会导致缓存重复的值。</p>
```java
import java.util.Map;
import java.util.concurrent.Callable;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.ExecutionException;
import java.util.concurrent.Future;
import java.util.concurrent.FutureTask;

public class Memozer3<A, V> implements Computable<A, V> {
	private final Map<A, Future<V>> cache = new ConcurrentHashMap<A, Future<V>>();
	private final Computable<A, V> c;

	public Memozer3(Computable<A, V> c) {
		this.c = c;
	}

	@Override
	public V compute(A arg) throws InterruptedException {
		Future<V> f = cache.get(arg);
		if (f == null) {
			Callable<V> eval = new Callable<V>() {
				@Override
				public V call() throws Exception {
					return c.compute(arg);
				}
			};
			FutureTask<V> ft = new FutureTask<V>(eval);
			f = ft;
			cache.put(arg, ft);
			ft.run();
		}
		V result = null;
		try {
			result = f.get();
		} catch (ExecutionException e) {
			e.printStackTrace();
		}
		return result;
	}
}
```
<p>Memozer3中，我们直接将任务缓存在容器中，如果已经存在（不管是否执行完成），则使用阻塞的方式获取任务结果。直到获取成功或抛出异常。但是这里的compute仍然会有重复计算的问题，尽管发生的概率比Memozer2小很多。</p>
```java
import java.util.Map;
import java.util.concurrent.Callable;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.ExecutionException;
import java.util.concurrent.Future;
import java.util.concurrent.FutureTask;

public class Memozer4<A, V> implements Computable<A, V> {
	private final Map<A, Future<V>> cache = new ConcurrentHashMap<A, Future<V>>();
	private final Computable<A, V> c;

	public Memozer4(Computable<A, V> c) {
		this.c = c;
	}

	@Override
	public V compute(A arg) throws InterruptedException {
		while (true) {
			Future<V> f = cache.get(arg);
			if (f == null) {
				Callable<V> eval = new Callable<V>() {
					@Override
					public V call() throws Exception {
						return c.compute(arg);
					}
				};
				FutureTask<V> ft = new FutureTask<V>(eval);
				f = cache.putIfAbsent(arg, ft);
				cache.put(arg, ft);
				ft.run();
			}
			V result = null;
			try {
				result = f.get();
			} catch (ExecutionException e) {
				e.printStackTrace();
			}
			return result;
		}
	}
}
```
<p>Memozer4在添加新的任务使用的是ConcurrentHashMap的putIfAbsent方法判断，利用了并发类的原子方法。避免了Memozer3的漏洞。</p>
<p>这样就完美了吗？并非如此。由于缓存的是任务，任务如果被取消，或执行抛出异常，会影响计算结果的获取。Memozer应该能够发现取消的计算，将其从容器中移除。如果任务抛出异常也应该将其从容器中移除。如果缓存过期，应该通过使用FutureTask的子类为每个缓存的结果指定一个过期时间，并定期扫描缓存中的过期元素。</p>
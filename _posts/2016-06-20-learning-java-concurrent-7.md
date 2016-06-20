---
date: 2016-06-20 22:16:12+08:00
layout: post
title: 一起学java并发编程（七）：性能和可伸缩性
thread: 9
categories: java
tags: java并发编程
---
<script type="text/javascript" src="http://cdn.mathjax.org/mathjax/latest/MathJax.js?config=default"></script>

### 性能
为了利用并发来实现更好的性能，我们需要努力做两件事：更有效地利用我们现有的处理资源，让我们的程序尽可能地开拓更多可用的处理资源。从性能监视器的视角来看，这意味着我们希望CPU尽可能处于忙碌状态。

可伸缩性指的是：当增加计算资源的时候（比如增加额外CPU数量、内存、存储器、I/O带宽），吞吐量和生产量能够相应地得以改进。

性能的这两个方面——“有多快”和“有多少”是完全分离的，有时候甚至是相悖的。为了实现更好的可伸缩性，或者更好地利用硬件，我们通常会停止增加每个独立任务所要完成的工作量，比如我们把任务分解到多个管道线的子任务中。具有讽刺意味的是，大多数在单线程化的程序中提高性能的窍门，都会损害可伸缩性。

从性能中的多个角度来看，“有多少”方面——可伸缩性，吞吐量和生产量——在Server应用程序中往往比“有多快”受更多的关注。

避免不成熟的优化。首先使程序正确，然后再加快——如果它运行得还不够快。

大多数性能的决定需要多个变量，并且高度依赖于发生的环境。在决定某个方案比其他方案“更快”之前，先问你自己一些问题：
- 你所谓的更“快”指的是什么？  
- 在什么样的条件下你的方案能够真正运行得更快？在轻负载还是重负载下？大数据集还是小数据集？是否支持你的测量标准的答案？  
- 这些条件在你的环境中发生的频率？是否支持你的测量标准的答案？  
- 这些代码在其他环境的不同条件下被用到的可能性？  
- 你用什么样隐含的代价，比如增加的开发风险或维护性，换取了性能的提高？这个权衡的决定是否正确？  

对性能的追求很可能是并发bug唯一最大的来源。

### Amdahl定律
Amdahl定律描述了在一个系统中，基于可并行化和串行化的组件各自所占的比重，程序通过获得额外的计算资源，理论上能够加速多少。如果F是必须串行化执行的比重，那么Amdahl定律告诉我们，在一个N处理器的机器中，我们最多可以加速：

$$ Speedup \leq \frac{1}{F + \frac{(1 - F)}{N}} $$

当N无效增大趋近无穷时，speedup的最大值无限趋近1/F，这意味着一个程序中如果50%的处理都需要串行进行的话，speedup只能提升2倍。

所有的并发程序都有一些串行源；如果你认为你没有，那么仔细检查吧。

减小锁的粒度，使用并发性良好的串行源。分离锁（把一个锁分拆成多个锁）和分拆锁（把一个锁分拆成两个锁）能够减小串行源锁的粒度，看上去没能在利用多处理器上帮助我们很多。但是实际效果却很好，因为分离出的数量可随着处理器数量的增加而增长。

### 线程开销
调度和线程内部的协调器都要付出性能的开销；对于性能改进的线程来说，并行带来的性能优势必须超过并发所引入的开销。

线程切换引入的JVM和OS活动的开销并不是切换上下文开销的唯一来源。当一个新的线程被换入后，它所需要的数据可能不在当前处理器本地的缓存中，所以切换上下文会引起缓存缺失的小恐慌，因此线程在第一次调度的时候会运行得稍慢一些。

如果线程频繁发生阻塞，那线程就不能完整使用它的调度限额了。一个程序发生越多的阻塞，与受限于CPU的程序相比，就会造成越多的上下文切换，这增加了调度的开销，并减少了吞吐量。

性能的开销有几个来源。synchronized和volatile提供的可见性保证要求使用一个特殊的、名为存储关卡的指令，来刷新缓存，使缓存无效，刷新硬件的写缓存，并延迟执行的传递。存储关卡可能同样会对性能产生影响，因为它们抑制了其他编译器的优化；在存储关卡中，大多数操作是不能被重排序的。synchronized机制对无竞争同步进行了优化（volatile总是非竞争的）。现代JVM能够通过优化，解除经确证不存在的锁，从而减少同步。更加成熟的JVM可以使用逸出分析来识别本地对象的引用并没有在堆中被暴露，并且因此成为线程本地的。即使没有逸出分析，编译器同样可以进行锁的粗化，把邻近的synchronized块用相同的锁合并起来。

不要过分担心非竞争的同步带来的开销，基础的机制已经足够快了，在这个基础上，JVM能够进行额外的优化，大大减少或消除了开销。关注那些真正发生了锁竞争的区域中性能的优化。

非竞争的同步可以由JVM完全掌握；而竞争的同步可能需要OS的活动，这会增大开销。当锁为竞争性的时候，失败的线程必然发生阻塞。JVM既能自旋等待，或者在操作系统中挂起这个被阻塞的线程。哪一个效率更高，取决于上下文切换的开销，以及成功地获取锁需要等待的时间这两者之间的关系。自旋等待更适合短期的等待，而挂起适合长时间等待。

### 减少锁竞争
串行化会损害可伸缩性，上下文切换会损害性能。竞争性的锁会同时导致这两种损失，所以减少所得竞争能够改进性能和可伸缩性。并发程序中，对可伸缩性首要的威胁是独占的资源锁。

有两个原因影响着锁的竞争性：锁被请求的频率，以及每次持有该锁的时间。

有三种方式来减少锁的竞争：
- 减少持有锁的时间；  
- 减少请求锁的频率；  
- 或者用协调机制取代独占锁，从而允许更强的并发性。  


### 减少上下文切换的开销
减小竞争发生的有效方式是尽可能缩短占有锁的时间。这可以通过把与锁无关的代码移出synchronized块来实现，尤其是那些花费“昂贵”的操作，以及那些潜在的阻塞操作，比如I/O操作。

持有锁超过必要范围的例子：
```java
import java.util.HashMap;
import java.util.Map;
import java.util.regex.Pattern;

public class AttributeStore {
	private final Map<String, String> attributes = new HashMap<String, String>();

	public synchronized boolean userLocationMatches(String name, String regexp) {
		String key = "users." + name + ".location";
		String location = attributes.get(key);
		if (location == null) {
			return false;
		} else {
			return Pattern.matches(regexp, location);
		}
	}
}
```
这里整个userLocationMatches方法都是synchronized的，实际上只有attributes.get(key)这一部分是真正需要调用锁的。
```java
import java.util.HashMap;
import java.util.Map;
import java.util.regex.Pattern;

public class BetterAttributeStore {
	private final Map<String, String> attributes = new HashMap<String, String>();

	public boolean userLocationMatches(String name, String regexp) {
		String key = "users." + name + ".location";
		String location;
		synchronized (this) {
			location = attributes.get(key);
		}
		if (location == null) {
			return false;
		} else {
			return Pattern.matches(regexp, location);
		}
	}
}
```
上面的代码我们缩小了锁守护的范围，减少了调用中遇到锁住情况的次数。串行化的代码少了，消除了可伸缩性的一个障碍。实际上如果我们把attributes换成线程安全的容器，能够省去显示的同步，减少Map访问中锁的范围，并减小了未来的维护者因为忘了在访问attributes前获得相应锁而造成的风险。

减小持有锁的时间比例的另一种方式是让线程减少调用它的频率。这可以通过分拆锁和分离锁来实现，也就是采用相互独立的锁，守卫多个独立的状态变量，在改变之前，它们都是由一个锁守护的。这些技术减小了锁发生的时的粒度，潜在实现了更好的可伸缩性——但是使用更多的锁同样会增加死锁的风险。

如果一个锁守护数量大于1、且相互独立的状态变量，你可以通过分拆锁，是每一个锁守护不同的变量，从而改进可伸缩性。
下面是一个使用默认的内置锁同步状态的例子：
```java
import java.util.Set;

public class ServerStatus {
	private final Set<String> users;
	private final Set<String> queries;

	public ServerStatus(Set<String> users, Set<String> queries) {
		this.users = users;
		this.queries = queries;
	}

	public synchronized void addUser(String u) {
		users.add(u);
	}

	public synchronized void addQuery(String q) {
		queries.add(q);
	}

	public synchronized void removeUser(String u) {
		users.remove(u);
	}

	public synchronized void removeQuery(String q) {
		queries.remove(q);
	}
}
```
默认的内置锁同步了两个独立变量的状态，我们可以分拆内置锁为两个独立的锁，分拆之后，每一个新的更精巧的锁，相比于那些原始的粗糙锁，将会看到更少的通信量。
```java
import java.util.Set;

public class ServerStatus {
	private final Set<String> users;
	private final Set<String> queries;

	public ServerStatus(Set<String> users, Set<String> queries) {
		this.users = users;
		this.queries = queries;
	}

	public void addUser(String u) {
		synchronized (users) {
			users.add(u);
		}
	}

	public void addQuery(String q) {
		synchronized (queries) {
			queries.add(q);
		}
	}

	public void removeUser(String u) {
		synchronized (users) {
			users.remove(u);
		}
	}

	public void removeQuery(String q) {
		synchronized (queries) {
			queries.remove(q);
		}
	}
}
```

分拆锁有时候可以被扩展，分成可大可小加锁块的集合，并且它们归属于相互独立的对象，这样的情况就是分离锁。例如，ConcurrentHashMap的实现使用了一个包含16个锁的Array，每个锁守护HashBucket的1/16；Bucket N有第N mod 16个锁来守护。

分离锁的一个负面作用是：对容器加锁，进行独占访问更加困难，并且更加昂贵了。

下面是一个分离锁的例子：
```java
public class StripedMap {
	private static final int N_LOCKS = 16;
	private final Node[] buckets;
	private final Object[] locks;

	private static class Node {
		...
	}

	public StripedMap(int numBuckets) {
		buckets = new Node[numBuckets];
		locks = new Object[N_LOCKS];
		for (int i = 0; i < N_LOCKS; i++) {
			locks[i] = new Object();
		}
	}

	private final int hash(Object key) {
		return Math.abs(key.hashCode() % buckets.length);
	}

	public Object get(Object key) {
		int hash = hash(key);
		synchronized (locks[hash % N_LOCKS]) {
			for (Node m = buckets[hash]; m != null; m = m.next()) {
				if (m.key.equals(key)) {
					return m.value;
				}
			}
		}
	}

	public void clear() {
		for (int i = 0; i < buckets.length; i++) {
			synchronized (locks[i % N_LOCKS]) {
				buckets[i] = null;
			}
		}
	}
	
	...
}
```
能够从分拆锁受益的程序，通常是那些锁的竞争普遍大于对锁守护数据竞争的程序。

要避免热点域。热点域（比如共享的缓存）会导致锁的粒度很难被降低。限制了可伸缩性。

用于减轻竞争锁带来的影响的第三种技术是提前使用独占锁，这有助于使用更友好的并发方式进行共享状态的管理。这包括使用并发容器、读-写锁、不可变对象，以及原子变量。

对象池化技术会给垃圾回收造成很大的麻烦。在并发的应用程序中，池化表现得更糟糕。当线程分配新的对象时，需要线程内部非常细微的协调，因为分配运算通常使用线程本地的分配块来消除对象堆中的大部分同步。但是，如果这些线程从池中请求对象，那么协调访问池的数据结构的同步就成为必然了，这边产生了线程阻塞的可能性。锁竞争产生的阻塞的代价比直接分配的代价多几百倍，即使很小的池竞争都会造成可伸缩性的瓶颈。

### 减少上下文的开销
在日志系统中，如果将I/O移入多线程程序中，会导致I/O资源竞争。只需要将日志I/O移入一个线程，既消除了输出流的竞争，又减少了竞争源，改进了整体的吞吐量。因为需要调度资源更少了，上下文切换更少了，锁的管理也更简单了。
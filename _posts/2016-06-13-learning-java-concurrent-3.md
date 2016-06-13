---
date: 2016-06-13 10:10:12+08:00
layout: post
title: 一起学java并发编程（三）：共享对象
thread: 5
categories: java
tags: java并发编程
---
### 可见性
<p>上一篇博客中我们提到，synchronized可以用于划定一个支持原子操作的“临界区”。实际上synchronized还具有一个重要、微妙的功能：内存可见性。内存可见性可以确保一个线程修改了对象状态之后，改变的状态能够被其他线程看到。通常，不能保证一个线程能及时读取其他线程写入的值，为了确保跨线程写入的内存可见性，就必须使用同步。</p>
```java
public class NoVisibility {
	private static boolean ready;
	private static int number;

	private static class ReaderThread extends Thread {
		@Override
		public void run() {
			while (!ready) {
				Thread.yield();
			}
			System.out.println(number);
		}
	}

	public static void main(String[] args) {
		new ReaderThread().start();
		number = 42;
		ready = true;
	}
}
```
<p>这段代码看起来没什么问题：直到ready为true才会打印number的值。但是有可能这段代码永远都不会终止，甚至由于编译器对指令“重排序”，导致输出的值为0。这是因为这里没有使用恰当的同步机制，不能保证主线程对ready和number修改的值能及时被ReadThread可见。</p>
> 关于重排序：在没有同步的情况下，编译器、处理器、运行时安排操作的执行顺序可能完全出人意料。在没有适当同步的多线程程序中，尝试推断那些“必然”发生在内存中的动作时，你总是会判断错误。

<p>在NoVisibility示例中，运行得到意外后果的原因是过期数据引起的：ReadThread检查ready时，有可能看到的是过期的数据。为了避免这种情况发生，每次访问变量都必须是同步的。这里有一个例子：</p>
```java
public class MutableInteger {
	private int value;

	public int getValue() {
		return value;
	}

	public void setValue(int value) {
		this.value = value;
	}
}
```
<p>如果有一个线程调用了setter，另一个线程正在调用getter，此时可能就看不到更新数据。我们需要通过同步getter和setter保证线程安全性。这里仅仅同步setter是不够的，因为getter仍然能够看见过期值。</p>
```java
public class SynchronizedInteger {
	private int value;

	public synchronized int getValue() {
		return value;
	}

	public synchronized void setValue(int value) {
		this.value = value;
	}
}
```
<p>一般来说，读取过期值如果是可接受的，对于原子变量可以不需要同步。但是java存储模型对于非volatile的long和double变量，JVM允许将64位的读写划分为两个32位的操作。如果读写不再同一线程，有可能读取了一个值的高32位和另一个值的低32位，此时获得的值完全不正确，需要将他们声明为volatile类型或用锁保护起来。</p>
<p>锁不仅仅关系到同步与互斥，也关系到内存可见性。为了保证所有线程都能看到共享的，可变变量的最新值，读取和写入线程必须使用公共的锁进行同步。</p>
<p>java提供了一种可见性同步的弱形式：volatile变量。当一个域声明为volatile后，编译器和运行时会监视这个变量：它是共享的，而且对它的操作不会与其他的内存操作一起被重排序。volatile变量不会缓存在寄存器或者缓存在对其他处理器隐藏的地方。volatile只关乎到可见性，访问volatile变量的操作不会加锁，也不会引起线程的阻塞，这使得volatile相对于synchronized而言，只是轻量级的同步机制。</p>
<p>只有当volatile变量能够简化实现和同步策略的验证时，才能使用它们。当验证正确性必须推断可见性问题时，应该避免使用volatile变量。正确使用volatile变量的方式包括：用于确保它们所引用的对象状态的可见性，或者用于标识重要的生命周期事件（比如初始化或关闭）的发生。</p>
<p>下面是一个volatile变量的典型应用：检查状态标记。</p>
```java
	volatile boolean asleep;
...
	while(!asleep){
		countSomeSheep();
	}
```
<p>加锁可以保证可见性与原子性；volatile变量只能保证可见性。只有满足下面所有的标准后，你才能使用volatile变量：</p>
- 写入变量时并不依赖变量的当前值；或者能够确保只有单一的线程修改变量的值；
- 变量不需要与其他的状态变量共同参与不变约束；
- 而且，访问变量时，没有其他的原因需要加锁。

### 发布与逸出
<p>发布一个对象的意思是使它能够被当前范围之外的代码所使用。一个对象在尚未准备好时就将它发布，这种情况称作逸出。</p>
<p>下面这个例子就是将一个HashSet实例通过knownSecrets发布的。</p>
```java
public static Set<Secret> knownSecrets;

public void initialize(){
	knownSecrets = new HashSet<Secret>();
}
```
<p>发布一个对象可能会间接地发布其他对象。如果将一个Secret对象加入集合knownSecrets中，发布了这个knownSecrets就已经发布了这个Secret对象。</p>
```java
class UnSafeStates{
	private String[] states=new String[]{"AK", "AL"};
	
	public String[] getStates(){
		return states;
	}
}
```
<p>以这种方式发布states会出问题，因为这种方式发布出去的states，任何调用者都能修改它，这个本应是私有的数据，事实上已经变成共有的了。</p>
<p>还有一种发布对象及其内部状态的机制是发布一个内部类实例。</p>
```java
public class ThisEscape{
	public ThisEscape(EventSource source){
		source.registerListener(
			new EventListener(){
				public void onEvent(Event e){
					doSomething(e);
				}
			}
		);
	}
}
```
<p>当ThisEscape发布EventListener时，也无条件地发布了封装ThisEscape的实例，因为内部类的实例包含了对封装实例的引用。这种情况隐式地允许this引用逸出。</p>
<p>另外，这种方式导致this在构造时逸出。从构造函数内发布的对象，只是一个未完成构造的对象，甚至在构造函数最后一行发布的引用也是如此。如果this引用在构造过程中逸出，这样的对象被认为是“没有正确构建的”。<strong>不要让this引用在构造期间逸出。</strong>在构造函数内启动一个线程，this引用几乎总是被创建的新线程共享，于是新线程在所属对象完成构造前就能看见它。在构造函数中创建线程并没有错误，但是最好不要立即启动它。应该发布一个start或者initialize方法来启动对象拥有的线程。</p>
```java
import java.awt.Event;
import java.util.EventListener;

public class SafeListener {
	private final EventListener listener;

	public SafeListener() {
		listener = new EventListener() {
			public void onEvent(Event e) {
				doSomething(e);
			}
		};
	}

	public static SafeListener newInstance(EventSource source){
		SafeListener safe=new SafeListener();
		source.registerListener(safe.listener);
		return safe;
	}
}
```

### 线程封闭
<p>线程封闭实质上就是不共享数据，这样就可以避免同步。</p>
<p>从设计实现上保证线程封闭称为ad-hoc线程限制，这不是一种正式的线程封闭技术，可以利用volatile变量实现特定的数据共享。只要确保只通过单一线程写入volatile变量，那么在这些volatile变量执行“读-改-写”操作就是安全的。（因为将修改限制在单一的线程中，从而阻止了竞争条件）。鉴于ad-hoc线程封闭固有的易损性，应该有节制地使用它。如果可能，应该使用线程封闭的强形式（栈限制或者ThreadLocal）取代它。</p>
#### 栈限制
<p>栈限制就是将对象封装在本地变量中，本地变量本身就被限制在执行线程中，它们存在于执行线程栈，其他线程无法访问这个栈。</p>
<p>下面是一个本地的基本类型和引用类型的变量的线程限制。</p>
```java
public int loadTheArk(Collection<Animal> candidates) {
	SortedSet<Animal> animals;
	int numPairs = 0;
	Animal candidate = null;
	// animals被限制在方法中，不要让它们逸出
	animals = new TreeSet<Animal>(new SpeciesGenderComparator());
	animals.addAll(candidates);
	for (Animal a : animals) {
		if (candidate == null || !candidate.isPotentialMate(a)) {
			candidate = a;
		} else {
			ark.load(new AnimalPair(candidate, a));
			++numPairs;
			candidate = null;
		}
	}
	return numPairs;
}
```

#### ThreadLocal
<p>ThreadLocal提供了get与set访问器，为每个使用它的线程维护一份单独的拷贝。所以get总是返回由当前执行线程通过set设置的最新值。实际上这就是一个线程级的缓存，与线程相关的值存储在线程对象自身中，线程终止后，这些值会被垃圾回收。在本系列第一篇博客中我们已经使用过ThreadLocal。但是如果全局变量，ThreadLocal变量会降低重用性，引入隐晦的类间耦合。应慎用。</p>

### 不可变性
<p><strong>不可变对象永远是线程安全的。</strong></p>
<p>只有满足如下状态，一个对象才是不可变的：</p>
- 它的状态不能在创建后再被修改；
- 所有域都是final类型；并且，
- 它被正确创建（创建期间没有发生this引用逸出）

<p>下面是一个构建于底层可变对象之上的不可变类</p>
```java
import java.util.HashSet;
import java.util.Set;

public final class ThreeStooges {
	private final Set<String> stooges = new HashSet<String>();

	public ThreeStooges() {
		stooges.add("Moe");
		stooges.add("Larry");
		stooges.add("Curly");
	}

	public boolean isStooge(String name) {
		return stooges.contains(name);
	}
}
```
<p>final在java存储模型中还有着特殊的语义。final域使得确保初始化安全性成为可能，初始化安全性让不可变对象不需要同步就能自由地被访问和共享。</p>
<p>将所有域声明为final类型，除非它们是可变的。</p>
```java
import java.util.Arrays;

public class OneValueCache {
	private final Integer lastNumber;
	private final Integer[] lastFactors;

	public OneValueCache(Integer lastNumber, Integer[] lastFactors) {
		this.lastNumber = lastNumber;
		this.lastFactors = lastFactors;
	}

	public Integer[] getFactors(Integer i) {
		if (lastNumber == null || !lastNumber.equals(i)) {
			return null;
		} else {
			return Arrays.copyOf(lastFactors, lastFactors.length);
		}
	}
}
```
<p>当一个线程设置volatile类型的cache域引用到一个新的OneValueCache后，新数据会立即对其他线程可见。与cache域相关的操作不会相互干扰，因为OneValueCache是不可变的，而且每次只有一条相应的代码路径访问它。</p>
```java
public class VolatileCachedFactorizer {
	private volatile OneValueCache cache = new OneValueCache(null, null);

	public void service(Integer i) {
		Integer[] factors = cache.getFactors(i);
		if (factors == null) {
			factors = factor(i);
			cache = new OneValueCache(i, factors);
		}
		send(factors);
	}
}
```

### 安全发布
<p>当我们希望跨线程共享对象时，我们必须安全地发布共享对象。</p>
#### 不正确的发布
<p>像下面这样简单地将一个对象的引用存储到公共域，并不足以发布一个对象。由于可见性问题，容器还是会在其他线程中被设置为一个不一致的状态。这种不一致状态会导致其他线程看到一个“部分创建的对象”。</p>
```java
public Holder holder;

public void initialize() {
	holder = new Holder(42);
}
```
Holde.java类
```java
public class Holder {
	private int n;
	
	public Holder(int n) {
		this.n = n;
	}
	
	public void assertSanity() {
		if (n != n) {
			throw new AssertionError("this statement is false.");
		}
	}
}
```
<p>这里没有同步来确保Holder对其他线程可见。这会导致两种问题：一是其他线程可能看见的是一个旧值；二是其他线程可能第一次读的是一个值（不是旧值，而是因为构造函数会调用父类的构造函数，父类会赋一个值），第二次读的是另一个值（Honder赋的一个值）。</p>

#### 不可变对象与初始化安全性
<p>不可变对象可以在没有额外同步的情况下，安全地用于任意线程；甚至发布它们时亦不需要同步。</p>
<p>上例中的Holder如果是不可变的，那么即使Holder没有正确的发布，assertSanity也不会抛出AssertionError。然而，如果final域指向可变对象，那么访问这些对象的状态时仍然需要同步。</p>

#### 安全发布的模式
<p>如果一个对象不是不可变的，它就必须被安全地发布，通常发布线程与消费线程都必须同步化。</p>
<p>为了安全地发布对象，对象的引用以及对象的状态必须同时对其他线程可见。一个正确创建的对象可以通过下列条件安全地发布：</p>
- 通过静态初始化器初始化对象的引用；
- 将它的引用存储到volatile域或AtomicRerence；
- 将它的引用存储到正确创建的对象的final域中；
- 或者将它的引用存储到由锁正确保护的域中。

<p>线程安全容器提供了如下的线程安全保证：</p>
- 置入Hashtable、synchronizedMap、ConcurrentMap中的主键以及键值，会安全地发布到可以从Map获得它们的任意线程中，无论是直接获得还是通过迭代器获得；
- 置入Vector、CopyOnWriteArrayList、CopyOnWriteArraySet、synchronizedList或者synchronizedSet中的元素，会安全地发布到可以从容器中获得它的任意线程中；
- 置入BlockingQueue或者ConcurrentLinkedQueue的元素，会安全地发布到可以从队列中获得它的任意线程中。

<p>通常，以最简单和最安全的方式发布一个被静态创建的对象，就是使用静态初始化器：</p>
```java
public static Holder holder = new Holder(42);
```

#### 高效不可变对象
<p>如果一下对象在发布后就不会再修改，只要能够保证这个对象“发布时”的状态对所有访问线程都可见，那么它的引用也都是可见的。这个对象称作有效不可变对象。</p>
<p>用有效不可变对象可以简化编程，并且由于减少了同步的使用，还能提高性能。</p>
<p>任何线程都可以在没有额外的同步下安全地使用一个安全发布的高效不可变对象。</p>

#### 可变对象
<p>发布对象的必要条件依赖于对象的可变性：</p>
- 不可变对象可以通过任意机制发布；
- 高效不可变对象必须要安全的发布；
- 可变对象必须要安全的发布，同时必须要线程安全或者被锁保护。

#### 安全地共享对象
<p>在并发程序中，使用和共享对象的一些最有效的策略如下：</p>
> 线程限制：一个线程限制的对象，通过限制在线程中，而被线程独占，且只能被占有它的线程修改。  
> 共享只读：一个共享的只读对象，在没有额外同步的情况下，可以被多个线程并发地访问，但是任何线程都不能修改它，只读共享对象包括可变对象与高效不可变对象。  
> 共享线程安全：一个线程安全的对象在内部进行同步，所以其他线程无须额外同步，就可以通过公共接口随意地访问它。  
> 被守护的：一个被守护的对象只能通过特定的锁来访问。被守护的对象包括那些被线程安全对象封装的对象，和已知被特定的锁保护起来的已发布对象。
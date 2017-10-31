---
date: 2016-06-14 10:10:12+08:00
layout: post
title: 一起学java并发编程（四）：组合对象
thread: 6
categories: java
tags: java并发编程
---
### 对象的状态
<p>设计线程安全类的过程应该包括下面 3 个基本要素：</p>
- 确定对象状态是由哪些变量构成的；  
- 确定限制状态变量的不变约束；  
- 制定一个管理并发访问对象状态的策略。  

<p>对象的状态首先与域有关。一个对象有n个基本域，它的状态就是域值组成的n元组。如果一个对象的域引用了其他对象，那么它的状态也同时包含了被引用对象的域。</p>
```java
public class Counter {
	private long value = 0;

	public synchronized long getValue() {
		return value;
	}

	public synchronized long increment() {
		if (value == Long.MAX_VALUE) {
			throw new IllegalStateException("counter overflow");
		}
		return ++value;
	}
}
```
<p>这里Counter的value是一个基本类型，这个value就构成Counter的完整状态。</p>
<p>同步策略定义了在不违背对象的不变约束的情况下，如何协调对其状态的访问。它规定了如何把不可变性、线程限制和锁结合起来，从而保护线程安全性，并指明哪些锁保护哪些变量。</p>
<p>对象与变量有一个状态空间：即它们可能处于的状态范围。状态空间越小，越容易判断它们。所以尽量使用final类型的域可以简化我们对对象的状态分析，因为它只有一个状态。</p>
<p>如果下一个状态源自当前状态，那么这个操作必须是复合操作。必须保证这种状态转换是安全的。不变约束与后验条件施加在状态及状态转换上的约束，引入了额外的同步与封装的需要。假设一个强制约束的类，如果某些状态可能是非法的，则必须封装，如果某个操作可能出现非法状态转换，则该操作必须是原子的。如果类没有被强制约束，则不需要强制保证封装与原子性。</p>
<p>不理解对象的不变约束和后验条件，就不能保证线程安全性。要约束状态变量的有效值或状态转换，就需要原子性与封装性。</p>
<p>单线程的先验条件很简单，要么成功要么失败。但是多线程的先验条件依赖于具体的处理场景，java提供了许多底层机制，将在以后说明。</p>
<p>当不变约束涉及多个变量时，任何一个操作在访问相关变量期间，线程必须占有保护这些变量的锁。</p>

### 实例限制
<p>实例限制就是用一个对象封装另一个对象。通过把限制与各种适当的锁策略相结合，可以确保程序以线程安全的方式使用其他非线程安全对象。</p>
<p>将数据封装在对象内部，把对数据的访问限制在对象方法上，更容易确保线程在访问数据时总能获得正确的锁。被限制对象一定不能逸出到它的期望可用范围之外。</p>
<p>下面就是一个实例限制的例子：</p>
```java
import java.util.HashSet;
import java.util.Set;

public class PersonSet {
	private final Set<Person> mySet = new HashSet<Person>();

	public synchronized void addPerson(Person p) {
		mySet.add(p);
	}

	public synchronized boolean containsPerson(Person p) {
		return mySet.contains(p);
	}
}
```
<p>尽管HashSet不是线程安全的，但是它被限制在PersonSet内部，不会逸出。能够访问mySet的代码都是线程安全的，因而确保了PersonSet的线程安全。</p>
<p>ArrayList 和 HashMap 这样的基本容器是非线程安全的，但是类库提供了包装器工厂方法，使这些非线程安全的类可以安全地用于多线程环境中。这些工厂方法利用 Decorator 模式，使用一个同步的包装器对象包装容器：包装器将相关接口的每个方法实现为同步方法，并将请求转发到下层的容器对象上。只要包装器对象占有着对下层容器唯一的可触及的引用，包装器对象就是线程安全的。</p>
<p>假设有一个获取车辆位置信息的程序：</p>
```java
import java.util.Collections;
import java.util.HashMap;
import java.util.Map;

public class MonitorVehicleTracker {
	private final Map<String, MutablePoint> locations;

	public MonitorVehicleTracker(Map<String, MutablePoint> locations) {
		this.locations = deepCopy(locations);
	}

	public synchronized Map<String, MutablePoint> getLocations() {
		return deepCopy(locations);
	}

	public synchronized MutablePoint getLocation(String id) {
		MutablePoint loc = locations.get(id);
		return loc == null ? null : new MutablePoint(loc);
	}

	public synchronized void setLocation(String id, int x, int y) {
		MutablePoint loc = locations.get(id);
		if (loc == null) {
			throw new IllegalArgumentException("No such ID: " + id);
		}
		loc.x = x;
		loc.y = y;
	}

	private Map<String, MutablePoint> deepCopy(Map<String, MutablePoint> m) {
		Map<String, MutablePoint> result = new HashMap<String, MutablePoint>();
		for (String id : m.keySet()) {
			result.put(id, new MutablePoint(m.get(id)));
		}
		return Collections.unmodifiableMap(result);
	}
}
```
```java
public class MutablePoint {
	public int x, y;

	public MutablePoint(MutablePoint mutablePoint) {
		this.x = mutablePoint.x;
		this.y = mutablePoint.y;
	}
}
```
<p>尽管MutablePoint不是线程安全的，但是MonitorVehicleTracker是。程序没有将locations发布出去（发布的是拷贝）。这个程序使用先复制可变数据，在返回给用户的方法维护者线程安全性。尽管存在着数据延迟、性能问题，但是它的确是线程安全的。</p>

### 委托线程安全性
<p>将不可变对象委托给线程安全的类。可以实现线程安全性。</p>
```java
import java.util.Collections;
import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.ConcurrentMap;

public class DelegatingVehicleTracker {
	private final ConcurrentMap<String, Point> locations;
	private final Map<String, Point> unmodifiableMap;

	public DelegatingVehicleTracker(Map<String, Point> point) {
		locations = new ConcurrentHashMap<String, Point>(point);
		unmodifiableMap = Collections.unmodifiableMap(locations);
	}

	public Map<String, Point> getLocations() {
		return unmodifiableMap;
	}

	public Point getLocation(String id) {
		return locations.get(id);
	}

	public void setLocation(String id, int x, int y) {
		if (locations.replace(id, new Point(x, y)) == null) {
			throw new IllegalArgumentException("invalid vehicle name: " + id);
		}
	}
}
```
```java
public class Point {
	public final int x, y;

	public Point(Point mutablePoint) {
		this.x = mutablePoint.x;
		this.y = mutablePoint.y;
	}

	public Point(int x, int y) {
		this.x = x;
		this.y = y;
	}
}
```
<p>这里Point是不可变的，所以不需要像上面的MutablePoint使用副本发布。</p>
<p>对于非状态依赖的变量都可以使用这种方式委托。</p>
```java
import java.awt.event.KeyListener;
import java.awt.event.MouseListener;
import java.util.List;
import java.util.concurrent.CopyOnWriteArrayList;

public class VisualComponent {
	private final List<KeyListener> keyListeners = new CopyOnWriteArrayList<KeyListener>();
	private final List<MouseListener> mouseListeners = new CopyOnWriteArrayList<MouseListener>();

	public void addKeyListener(KeyListener listener) {
		keyListeners.add(listener);
	}

	public void addMouseListener(MouseListener listener) {
		mouseListeners.add(listener);
	}

	public void removeKeyListener(KeyListener listener) {
		keyListeners.remove(listener);
	}

	public void removeMouseListener(MouseListener listener) {
		mouseListeners.remove(listener);
	}
}
```
<p>但是如果一个类中的变量有状态依赖关系，类必须提供它自有的锁以保证复合操作的原子性。除非整个复合操作也可以委托给一个状态变量。</p>
<p>如果一个类由多个彼此独立的线程安全的状态变量组成，并且类的操作不包含任何无效状态转换时，可以将线程安全委托给这些状态变量。</p>
<p>如果一个状态变量是线程安全的，没有任何不变的约束限制它的值，并且没有任何状态转换限制它的操作，那么它可以被安全发布。</p>
<p>假如我们将上面交通工具位置管理的Point改为可变的，那么我们就需要保证Point必须是线程安全的。我们创建一个SafePoint，获取和设置SafePoint中x，y的值必须是原子的。</p>
```java
public class SafePoint {
	private int x, y;

	private SafePoint(int[] a) {
		this(a[0], a[1]);
	}

	private SafePoint(int x, int y) {
		this.x = x;
		this.y = y;
	}

	public SafePoint(SafePoint p) {
		this(p.get());
	}

	private synchronized int[] get() {
		return new int[] { x, y };
	}

	public synchronized void set(int x, int y) {
		this.x = x;
		this.y = y;
	}
}
```
```java
import java.util.Collections;
import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;

public class PublishingVehicleTracker {
	private final Map<String, SafePoint> locations;
	private final Map<String, SafePoint> unmodifiableMap;

	public PublishingVehicleTracker(Map<String, SafePoint> locations) {
		this.locations = new ConcurrentHashMap<String, SafePoint>(locations);
		this.unmodifiableMap = Collections.unmodifiableMap(this.locations);
	}

	public Map<String, SafePoint> getLocations() {
		return unmodifiableMap;
	}

	public SafePoint getLocation(String id) {
		return unmodifiableMap.get(id);
	}

	public void setLocation(String id, int x, int y) {
		if (!locations.containsKey(id)) {
			throw new IllegalArgumentException("invalid vehicle name: " + id);
		}
		locations.get(id).set(x, y);
	}
}
```
PublishingVehicleTracker没有给location添加任何约束，所以不需要同步，否则也必须同步。

### 向已有的线程安全类添加功能
<p>有时候我们需要的线程安全功能可能大部分已经存在于一个已有的线程安全类了，我们有两种方式将其扩展成符合我们要求的类：一种是修改源码，另一种是扩展这个类。通常我们并没有修改源码的权限。此时我们就需要保证扩展类的同步策略不会破坏已有类的同步策略。</p>
<p>我们将Vector扩展一个“缺少即加入”的功能。这里使用的是扩展类的方式（继承）。</p>
```java
public class BetterVector<E> extends Vector<E> {
	public synchronized boolean putIfAbsent(E e) {
		boolean absent = !contains(e);
		if (absent) {
			add(e);
		}
		return absent;
	}
}
```
<p>对于一个由Collections.synchronizedList封装的ArrayList，需要采用扩展功能的方式（包装类），因为客户代码甚至不知道同步封装工厂方法返回的List对象的类型。</p>
```java
import java.util.ArrayList;
import java.util.Collections;
import java.util.List;

public class ListHelper<E> {
	public List<E> list = Collections.synchronizedList(new ArrayList<E>());

	public synchronized boolean putIfAbsent(E e) {
		boolean absent = !list.contains(e);
		if (absent) {
			list.add(e);
		}
		return absent;
	}
}
```
<p>上面这段代码是错误的！尽管它使用synchronized同步了putIfAbsent操作，但是无论List使用哪个锁保护它的状态，可以确定的是这个锁并没有用到ListHelper上。由于不同的锁，所以putIfAbsent的操作并不是原子化的。</p>
<p>为了保证这个方法能正确的工作我们必须保证方法锁使用的锁与List用于客户端加锁与外部加锁时所使用的锁是同一个锁。</p>
```java
import java.util.ArrayList;
import java.util.Collections;
import java.util.List;

public class ListHelper<E> {
	public List<E> list = Collections.synchronizedList(new ArrayList<E>());

	public boolean putIfAbsent(E e) {
		synchronized (list) {
			boolean absent = !list.contains(e);
			if (absent) {
				list.add(e);
			}
			return absent;
		}
	}
}
```
<p>如此，使用的就是list对象锁对这个方法进行了加锁，保证了线程同步。</p>
<p>其实还有更好的方法来处理这种情况：组合。</p>
```java
import java.util.List;

public class ImprovedList<T> {
	private final List<T> list;

	public ImprovedList(List<T> list) {
		this.list = list;
	}

	public synchronized boolean putIfAbsent(T t) {
		boolean contains = list.contains(t);
		if (!contains) {
			list.add(t);
		}
		return !contains;
	}

	public synchronized void clear() {
		list.clear();
	}
}
```
<p>ImprovedList类将list封装起来，引入了一个新的锁层。我们只在这里提供了底层List的唯一外部引用。</p>
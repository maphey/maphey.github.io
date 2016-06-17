---
date: 2016-06-17 22:16:12+08:00
layout: post
title: 一起学java并发编程（六）：活跃度危险
thread: 8
categories: java
tags: java并发编程
---
<p>活跃度失败是非常严重的问题，除了中断应用程序，没有任何机制能恢复这种失败。</p>

### 死锁
<p>当一个线程永远占有一个锁，而其他线程尝试获取这个锁，它们永远会被阻塞。这就是死锁。</p>
<p>内部调用死锁的例子：</p>
```java
public class LeftRightDeadLock {
	private final Object left = new Object();
	private final Object right = new Object();

	public void leftRight() {
		synchronized (left) {
			synchronized (right) {
				doSomething();
			}
		}
	}

	public void rightLeft() {
		synchronized (right) {
			synchronized (left) {
				doSomething();
			}
		}
	}
}
```
<p>上面代码试图获取两个相同的锁，但是获取顺序却不一致，所以发生了死锁。如果获取锁的顺序相同，就不会出现循环的锁依赖现象，也不会发生死锁了。</p>
<p>如果所有线程以通用的固定顺序获取锁，程序就不会出现锁顺序死锁问题了。</p>
<p>外部引用导致死锁的例子：</p>
```java
public void transferMoney(Account fromAccount, Account toAccount, DollarAmount amount) throws InsufficientFundsException {
	synchronized (fromAccount) {
		synchronized (toAccount) {
			if (fromAccount.getBalance().compareTo(amount) < 0) {
				throw new InsufficientFundsException();
			} else {
				fromAccount.debit(amount);
				toAccount.credit(amount);
			}
		}
	}
}
```
<p>上面的例子看起来没有问题，但是也依赖于外部传递的对象的顺序。如果有两个线程的fromAccount和toAccount刚好相反，就会发生死锁。</p>
<p>在指定锁的顺序时，可以使用System.identityHashCode这样一种方式，它会返回Object.hashCode的值。在极少数情况下，两个对象可能具有相同的hashCode，我们必须使用任意的“中数”来决定锁的顺序，为了解决这种死锁问题，我们引入裁决锁（tie-breaking）,在获取这两个锁之前，先要获得裁决锁这样我们就能保证只有一个线程以未知的顺序获取锁。</p>
<p>对加锁顺序进行限制以解决死锁的例子：</p>
```java
private static final Object tieLock = new Object();

public void transferMoney(final Account fromAcct, final Account toAcct, final DollarAmount amount) throws InsufficientFundsException {
	class Helper {
		public void transfer() {
			if (fromAcct.getBalance().compareTo(amount) < 0) {
				throw new InsufficientFundsException();
			} else {
				fromAcct.debit(amount);
				toAcct.credit(amount);
			}
		}
	}

	int fromHash = System.identityHashCode(fromAcct);
	int toHash = System.identityHashCode(toAcct);

	if (fromHash < toHash) {
		synchronized (fromAcct) {
			synchronized (toAcct) {
				new Helper().transfer();
			}
		}
	} else if (fromHash > toHash) {
		synchronized (toAcct) {
			synchronized (fromAcct) {
				new Helper().transfer();
			}
		}
	} else {
		synchronized (tieLock) {
			synchronized (fromAcct) {
				synchronized (toAcct) {
					new Helper().transfer();
				}
			}
		}
	}
}
```
<p>上面的代码就使用了第三个tieLock作为裁决锁。如果账号具有唯一的key值，直接用key值判断锁的顺序就更容易了。</p>
<p>很多时候导致死锁的获取锁顺序并不明显，那些协作对象间的锁获取比较隐蔽，也有可能因为锁顺序问题产生死锁。</p>
<p>协作间对象导致死锁的例子：</p>
```java
class Taxi {
	private Point location, destination;
	private final Dispatcher dispatcher;

	public Taxi(Dispatcher dispatcher) {
		this.dispatcher = dispatcher;
	}

	public synchronized Point getLocation() {
		return location;
	}

	public synchronized void setLocation(Point location) {
		this.location = location;
		if (location.equals(destination)) {
			dispatcher.notifyAvailable(this);
		}
	}
}

class Dispatcher {
	private final Set<Taxi> taxis;
	private final Set<Taxi> availableTaxis;

	public Dispatcher() {
		this.taxis = new HashSet<Taxi>();
		this.availableTaxis = new HashSet<Taxi>();
	}

	public synchronized void notifyAvailable(Taxi taxi) {
		availableTaxis.add(taxi);
	}

	public synchronized Image getImage() {
		Image image = new Image();
		for (Taxi t : availableTaxis) {
			image.drawMarker(t.getLocation());
		}
		return image;
	}
}
```
<p>Dispatcher中的getImage会调用Taxi的getLocation方法，于是先获取了Dispatcher的内置锁，接着获取了Taxi的内置锁；Taxi的setLocation和getLocation使用的都是同一个内置锁，Taxi的setLocation先获取了Taxi的内置锁，接着获取了Dispatcher的内置锁。如果有两个线程同时调用Dispatcher的getImage和Taxi的setLocation，就会因为加锁顺序问题而死锁。</p>
<p>在持有锁的时候调用外部方法是在挑战活跃度问题。外部方法可能会获得其他锁（产生死锁的风险），或者遭遇严重超时的阻塞。当你持有锁的时候会延迟其他试图获得该锁的线程。在持有锁的时候调用一个外部方法很难进行分析，因此是危险的。</p>
<p>当调用的方法不需要持有锁时，这被称为开放调用，依赖于开放调用的类会具有更好的行为，并且比那些需要获得锁才能调用的方法相比，更容易与其他的类合作。</p>
<p>使用开放调用避免协作间对象调用产生的死锁：</p>
```java
class Taxi {
	private Point location, destination;
	private final Dispatcher dispatcher;

	public Taxi(Dispatcher dispatcher) {
		this.dispatcher = dispatcher;
	}

	public synchronized Point getLocation() {
		return location;
	}

	public void setLocation(Point location) {
		boolean reachedDestination;
		synchronized (this) {
			this.location = location;
			reachedDestination = location.equals(destination);
		}
		if (reachedDestination) {
			dispatcher.notifyAvailable(this);
		}
	}
}

class Dispatcher {
	private final Set<Taxi> taxis;
	private final Set<Taxi> availableTaxis;

	public Dispatcher() {
		this.taxis = new HashSet<Taxi>();
		this.availableTaxis = new HashSet<Taxi>();
	}

	public synchronized void notifyAvailable(Taxi taxi) {
		availableTaxis.add(taxi);
	}

	public Image getImage() {
		Set<Taxi> copy;
		synchronized (this) {
			copy = new HashSet<Taxi>(taxis);
		}
		Image image = new Image();
		for (Taxi t : copy) {
			image.drawMarker(t.getLocation());
		}
		return image;
	}
}
```
<p>将之前协作间对象调用的代码使用开放调用的方法修改，只同步那些调用共享状态的操作，而不是将整个方法用锁保护起来。Taxi和Dispatcher不再在持有自身锁的情况下访问外部方法，实现了开放调用。</p>
<p>在程序中尽量使用开放调用，依赖于开放调用的程序，相比于那些在持有锁的时候还调用外部方法的程序，更容易进行死锁自由度的分析。</p>

### 避免和诊断死锁
<p>如果一个程序一次至多获得一个锁，那么就不会产生锁顺序死锁。当然，这通常不现实。但是如果能够避免这种情况，就能够省去很多工作。如果你必须获得多个锁，那么锁的顺序必须是你设计工作的一部分：尽量减少潜在锁之间的交互数量，遵守并文档化该锁顺序协议，这些缺一不可。</p>
<p>显式锁有一个定时tryLock特性，在规定的时间之后tryLock还没有获得锁就返回失败。</p>
<p>如果发生了死锁，可以使用JVM的线程转储来帮助你识别死锁的发生。在Unix平台，你可以通过向JVM进程发送SIGQIUT信号（kill-3）来触发线程转储。</p>

### 其他活跃度危险
<h4>饥饿</h4>
<p>饥饿是由于永远获取不到需要的资源导致的。多线程中饥饿的表现就是线程永远得不到CPU资源。一般情况下，线程优先级是导致线程饥饿的主要原因。</p>
<p>抵制使用线程优先级的诱惑，因为这会增加平台依赖性，并且可能引起活跃度问题。大多数并发应用程序可以对所有线程使用相同的优先级。</p>

<h4>弱响应性</h4>
<p>如果后台线程是耗CPU资源的线程，可能会影响界面线程的响应性。这时应该降低后台线程的优先级。</p>
<p>不良的锁管理也可能导致弱响应性。一个线程长期持有锁，导致其他线程必须等待。</p>

<h4>活锁</h4>
<p>如果线程没有被阻塞，但是不能继续进行，只能不断的重试，就导致了活锁。活锁通常发生在消息处理队列中。由于一个消息发送失败导致不断重试，阻塞了后面消息的发送。我们可以设置重试次数，当失败超过重试次数就将失败的元素从队列中移除，并将错误信息持久化入库。</p>
<p>多个互相协作的线程，当它们为了彼此间响应而修改了状态，使得没有一个线程能够继续前进，那么就发生了活锁。解决这种活锁问题可以对重试机制引入一些随机性，通过随机等待和撤回进行重试。</p>
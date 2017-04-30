---
title: Java并发编程：Lock
date: 2017-04-30 22:03:57
categories: Java多线程系列
tags: [java多线程]
description: Lock和synchronized有一点非常大的不同，采用synchronized不需要用户去手动释放锁，当synchronized方法或者synchronized代码块执行完之后，系统会自动让线程释放对锁的占用；而Lock则必须要用户去手动释放锁，如果没有主动释放锁，就有可能导致出现死锁现象
---
在上一篇文章中我们讲到了如何使用关键字synchronized来实现同步访问。本文我们继续来探讨这个问题，从Java 5之后，在java.util.concurrent.locks包下提供了另外一种方式来实现同步访问，那就是Lock。

　　也许有朋友会问，既然都可以通过synchronized来实现同步访问了，那么为什么还需要提供Lock？这个问题将在下面进行阐述。本文先从synchronized的缺陷讲起，然后再讲述java.util.concurrent.locks包下常用的有哪些类和接口，最后讨论以下一些关于锁的概念方面的东西

　　以下是本文目录大纲：

　　一.synchronized的缺陷

　　二.java.util.concurrent.locks包下常用的类

　　三.锁的相关概念介绍

　　若有不正之处请多多谅解，并欢迎批评指正。

　　请尊重作者劳动成果，转载请标明原文链接：

 　　http://www.cnblogs.com/dolphin0520/p/3923167.html

## 一.synchronized的缺陷

　　synchronized是java中的一个关键字，也就是说是Java语言内置的特性。那么为什么会出现Lock呢？

　　在上面一篇文章中，我们了解到如果一个代码块被synchronized修饰了，当一个线程获取了对应的锁，并执行该代码块时，其他线程便只能一直等待，等待获取锁的线程释放锁，而这里获取锁的线程释放锁只会有两种情况：

　　1）获取锁的线程执行完了该代码块，然后线程释放对锁的占有；

　　2）线程执行发生异常，此时JVM会让线程自动释放锁。

　　那么如果这个获取锁的线程由于要等待IO或者其他原因（比如调用sleep方法）被阻塞了，但是又没有释放锁，其他线程便只能干巴巴地等待，试想一下，这多么影响程序执行效率。

　　因此就需要有一种机制可以不让等待的线程一直无期限地等待下去（比如只等待一定的时间或者能够响应中断），通过Lock就可以办到。

　　再举个例子：当有多个线程读写文件时，读操作和写操作会发生冲突现象，写操作和写操作会发生冲突现象，但是读操作和读操作不会发生冲突现象。

　　但是采用synchronized关键字来实现同步的话，就会导致一个问题：

　　如果多个线程都只是进行读操作，所以当一个线程在进行读操作时，其他线程只能等待无法进行读操作。

　　因此就需要一种机制来使得多个线程都只是进行读操作时，线程之间不会发生冲突，通过Lock就可以办到。

　　另外，通过Lock可以知道线程有没有成功获取到锁。这个是synchronized无法办到的。

　　总结一下，也就是说Lock提供了比synchronized更多的功能。但是要注意以下几点：

　　1）Lock不是Java语言内置的，synchronized是Java语言的关键字，因此是内置特性。Lock是一个类，通过这个类可以实现同步访问；

　　2）Lock和synchronized有一点非常大的不同，采用synchronized不需要用户去手动释放锁，当synchronized方法或者synchronized代码块执行完之后，系统会自动让线程释放对锁的占用；而Lock则必须要用户去手动释放锁，如果没有主动释放锁，就有可能导致出现死锁现象。

## 二.java.util.concurrent.locks包下常用的类

　　下面我们就来探讨一下java.util.concurrent.locks包中常用的类和接口。

　　1.Lock

　　首先要说明的就是Lock，通过查看Lock的源码可知，Lock是一个接口：  

	public interface Lock {
		void lock();
		void lockInterruptibly() throws InterruptedException;
		boolean tryLock();
		boolean tryLock(long time, TimeUnit unit) throws InterruptedException;
		void unlock();
		Condition newCondition();
	}
 　　下面来逐个讲述Lock接口中每个方法的使用，lock()、tryLock()、tryLock(long time, TimeUnit unit)和lockInterruptibly()是用来获取锁的。unLock()方法是用来释放锁的。newCondition()这个方法暂且不在此讲述，会在后面的线程协作一文中讲述。

　　在Lock中声明了四个方法来获取锁，那么这四个方法有何区别呢？

　　首先lock()方法是平常使用得最多的一个方法，就是用来获取锁。如果锁已被其他线程获取，则进行等待。

　　由于在前面讲到如果采用Lock，必须主动去释放锁，并且在发生异常时，不会自动释放锁。因此一般来说，使用Lock必须在try{}catch{}块中进行，并且将释放锁的操作放在finally块中进行，以保证锁一定被被释放，防止死锁的发生。通常使用Lock来进行同步的话，是以下面这种形式去使用的：

	Lock lock = ...;
	lock.lock();
	try{
		//处理任务
	}catch(Exception ex){
		 
	}finally{
		lock.unlock();   //释放锁
	}
　　tryLock()方法是有返回值的，它表示用来尝试获取锁，如果获取成功，则返回true，如果获取失败（即锁已被其他线程获取），则返回false，也就说这个方法无论如何都会立即返回。在拿不到锁时不会一直在那等待。

　　tryLock(long time, TimeUnit unit)方法和tryLock()方法是类似的，只不过区别在于这个方法在拿不到锁时会等待一定的时间，在时间期限之内如果还拿不到锁，就返回false。如果如果一开始拿到锁或者在等待期间内拿到了锁，则返回true。

　　所以，一般情况下通过tryLock来获取锁时是这样使用的:  

	Lock lock = ...;
	if(lock.tryLock()) {
		 try{
			 //处理任务
		 }catch(Exception ex){
			 
		 }finally{
			 lock.unlock();   //释放锁
		 } 
	}else {
		//如果不能获取锁，则直接做其他事情
	}
 　　lockInterruptibly()方法比较特殊，当通过这个方法去获取锁时，如果线程正在等待获取锁，则这个线程能够响应中断，即中断线程的等待状态。也就使说，当两个线程同时通过lock.lockInterruptibly()想获取某个锁时，假若此时线程A获取到了锁，而线程B只有在等待，那么对线程B调用threadB.interrupt()方法能够中断线程B的等待过程。

　　由于lockInterruptibly()的声明中抛出了异常，所以lock.lockInterruptibly()必须放在try块中或者在调用lockInterruptibly()的方法外声明抛出InterruptedException。

　　因此lockInterruptibly()一般的使用形式如下：

	public void method() throws InterruptedException {
		lock.lockInterruptibly();
		try {  
		 //.....
		}
		finally {
			lock.unlock();
		}  
	}
　　注意，当一个线程获取了锁之后，是不会被interrupt()方法中断的。因为本身在前面的文章中讲过单独调用interrupt()方法不能中断正在运行过程中的线程，只能中断阻塞过程中的线程。

　　因此当通过lockInterruptibly()方法获取某个锁时，如果不能获取到，只有进行等待的情况下，是可以响应中断的。

　　而用synchronized修饰的话，当一个线程处于等待某个锁的状态，是无法被中断的，只有一直等待下去。

　　2.ReentrantLock

　　ReentrantLock，意思是“可重入锁”，关于可重入锁的概念在下一节讲述。ReentrantLock是唯一实现了Lock接口的类，并且ReentrantLock提供了更多的方法。下面通过一些实例看具体看一下如何使用ReentrantLock。

　　例子1，lock()的正确使用方法

	public class Test {
		private ArrayList<Integer> arrayList = new ArrayList<Integer>();
		public static void main(String[] args)  {
			final Test test = new Test();
			 
			new Thread(){
				public void run() {
					test.insert(Thread.currentThread());
				};
			}.start();
			 
			new Thread(){
				public void run() {
					test.insert(Thread.currentThread());
				};
			}.start();
		}  
     
    public void insert(Thread thread) {
        Lock lock = new ReentrantLock();    //注意这个地方
        lock.lock();
        try {
            System.out.println(thread.getName()+"得到了锁");
            for(int i=0;i<5;i++) {
                arrayList.add(i);
            }
        } catch (Exception e) {
            // TODO: handle exception
        }finally {
            System.out.println(thread.getName()+"释放了锁");
            lock.unlock();
        }
    }
	}
 　　各位朋友先想一下这段代码的输出结果是什么？


	Thread-0得到了锁
	Thread-1得到了锁
	Thread-0释放了锁
	Thread-1释放了锁
　　也许有朋友会问，怎么会输出这个结果？第二个线程怎么会在第一个线程释放锁之前得到了锁？原因在于，在insert方法中的lock变量是局部变量，每个线程执行该方法时都会保存一个副本，那么理所当然每个线程执行到lock.lock()处获取的是不同的锁，所以就不会发生冲突。

　　知道了原因改起来就比较容易了，只需要将lock声明为类的属性即可。 

	public class Test {
		private ArrayList<Integer> arrayList = new ArrayList<Integer>();
		private Lock lock = new ReentrantLock();    //注意这个地方
		public static void main(String[] args)  {
			final Test test = new Test();
			 
			new Thread(){
				public void run() {
					test.insert(Thread.currentThread());
				};
			}.start();
			 
			new Thread(){
				public void run() {
					test.insert(Thread.currentThread());
				};
			}.start();
		}  
		 
		public void insert(Thread thread) {
			lock.lock();
			try {
				System.out.println(thread.getName()+"得到了锁");
				for(int i=0;i<5;i++) {
					arrayList.add(i);
				}
			} catch (Exception e) {
				// TODO: handle exception
			}finally {
				System.out.println(thread.getName()+"释放了锁");
				lock.unlock();
			}
		}
	}
 　　这样就是正确地使用Lock的方法了。

　　例子2，tryLock()的使用方法

	public class Test {
		private ArrayList<Integer> arrayList = new ArrayList<Integer>();
		private Lock lock = new ReentrantLock();    //注意这个地方
		public static void main(String[] args)  {
			final Test test = new Test();
			 
			new Thread(){
				public void run() {
					test.insert(Thread.currentThread());
				};
			}.start();
			 
			new Thread(){
				public void run() {
					test.insert(Thread.currentThread());
				};
			}.start();
		}  
		 
		public void insert(Thread thread) {
			if(lock.tryLock()) {
				try {
					System.out.println(thread.getName()+"得到了锁");
					for(int i=0;i<5;i++) {
						arrayList.add(i);
					}
				} catch (Exception e) {
					// TODO: handle exception
				}finally {
					System.out.println(thread.getName()+"释放了锁");
					lock.unlock();
				}
			} else {
				System.out.println(thread.getName()+"获取锁失败");
			}
		}
	}
 　　输出结果：

	Thread-0得到了锁
	Thread-1获取锁失败
	Thread-0释放了锁
	
　　例子3，lockInterruptibly()响应中断的使用方法：

	public class Test {
		private Lock lock = new ReentrantLock();   
		public static void main(String[] args)  {
			Test test = new Test();
			MyThread thread1 = new MyThread(test);
			MyThread thread2 = new MyThread(test);
			thread1.start();
			thread2.start();
			 
			try {
				Thread.sleep(2000);
			} catch (InterruptedException e) {
				e.printStackTrace();
			}
			thread2.interrupt();
		}  
		 
		public void insert(Thread thread) throws InterruptedException{
			lock.lockInterruptibly();   //注意，如果需要正确中断等待锁的线程，必须将获取锁放在外面，然后将InterruptedException抛出
			try {  
				System.out.println(thread.getName()+"得到了锁");
				long startTime = System.currentTimeMillis();
				for(    ;     ;) {
					if(System.currentTimeMillis() - startTime >= Integer.MAX_VALUE)
						break;
					//插入数据
				}
			}
			finally {
				System.out.println(Thread.currentThread().getName()+"执行finally");
				lock.unlock();
				System.out.println(thread.getName()+"释放了锁");
			}  
		}
	}
 
	class MyThread extends Thread {
		private Test test = null;
		public MyThread(Test test) {
			this.test = test;
		}
		@Override
		public void run() {
			 
			try {
				test.insert(Thread.currentThread());
			} catch (InterruptedException e) {
				System.out.println(Thread.currentThread().getName()+"被中断");
			}
		}
	}
　　运行之后，发现thread2能够被正确中断。

　　3.ReadWriteLock

　　ReadWriteLock也是一个接口，在它里面只定义了两个方法：

	public interface ReadWriteLock {
		/**
		 * Returns the lock used for reading.
		 *
		 * @return the lock used for reading.
		 */
		Lock readLock();
	 
		/**
		 * Returns the lock used for writing.
		 *
		 * @return the lock used for writing.
		 */
		Lock writeLock();
	}
 　　一个用来获取读锁，一个用来获取写锁。也就是说将文件的读写操作分开，分成2个锁来分配给线程，从而使得多个线程可以同时进行读操作。下面的ReentrantReadWriteLock实现了ReadWriteLock接口。

　　4.ReentrantReadWriteLock

　　ReentrantReadWriteLock里面提供了很多丰富的方法，不过最主要的有两个方法：readLock()和writeLock()用来获取读锁和写锁。

　　下面通过几个例子来看一下ReentrantReadWriteLock具体用法。

　　假如有多个线程要同时进行读操作的话，先看一下synchronized达到的效果：

	public class Test {
		private ReentrantReadWriteLock rwl = new ReentrantReadWriteLock();
		 
		public static void main(String[] args)  {
			final Test test = new Test();
			 
			new Thread(){
				public void run() {
					test.get(Thread.currentThread());
				};
			}.start();
			 
			new Thread(){
				public void run() {
					test.get(Thread.currentThread());
				};
			}.start();
			 
		}  
		 
		public synchronized void get(Thread thread) {
			long start = System.currentTimeMillis();
			while(System.currentTimeMillis() - start <= 1) {
				System.out.println(thread.getName()+"正在进行读操作");
			}
			System.out.println(thread.getName()+"读操作完毕");
		}
	}
 　　这段程序的输出结果会是，直到thread1执行完读操作之后，才会打印thread2执行读操作的信息。

	Thread-0正在进行读操作
	Thread-0正在进行读操作
	Thread-0正在进行读操作
	Thread-0正在进行读操作
	Thread-0正在进行读操作
	Thread-0正在进行读操作
	Thread-0正在进行读操作
	Thread-0正在进行读操作
	Thread-0正在进行读操作
	Thread-0正在进行读操作
	Thread-0正在进行读操作
	Thread-0正在进行读操作
	Thread-0正在进行读操作
	Thread-0正在进行读操作
	Thread-0正在进行读操作
	Thread-0正在进行读操作
	Thread-0正在进行读操作
	Thread-0正在进行读操作
	Thread-0正在进行读操作
	Thread-0正在进行读操作
	Thread-0正在进行读操作
	Thread-0正在进行读操作
	Thread-0正在进行读操作
	Thread-0正在进行读操作
	Thread-0正在进行读操作
	Thread-0正在进行读操作
	Thread-0正在进行读操作
	Thread-0正在进行读操作
	Thread-0读操作完毕
	Thread-1正在进行读操作
	Thread-1正在进行读操作
	Thread-1正在进行读操作
	Thread-1正在进行读操作
	Thread-1正在进行读操作
	Thread-1正在进行读操作
	Thread-1正在进行读操作
	Thread-1正在进行读操作
	Thread-1正在进行读操作
	Thread-1正在进行读操作
	Thread-1正在进行读操作
	Thread-1正在进行读操作
	Thread-1正在进行读操作
	Thread-1正在进行读操作
	Thread-1正在进行读操作
	Thread-1正在进行读操作
	Thread-1正在进行读操作
	Thread-1正在进行读操作
	Thread-1正在进行读操作
	Thread-1正在进行读操作
	Thread-1正在进行读操作
	Thread-1正在进行读操作
	Thread-1正在进行读操作
	Thread-1正在进行读操作
	Thread-1正在进行读操作
	Thread-1正在进行读操作
	Thread-1正在进行读操作
	Thread-1正在进行读操作
	Thread-1正在进行读操作
	Thread-1正在进行读操作
	Thread-1正在进行读操作
	Thread-1正在进行读操作
	Thread-1正在进行读操作
	Thread-1正在进行读操作
	Thread-1正在进行读操作
	Thread-1正在进行读操作
	Thread-1正在进行读操作
	Thread-1正在进行读操作
	Thread-1正在进行读操作
	Thread-1正在进行读操作
	Thread-1正在进行读操作
	Thread-1正在进行读操作
	Thread-1正在进行读操作
	Thread-1读操作完毕  
	
　　而改成用读写锁的话：

	public class Test {
		private ReentrantReadWriteLock rwl = new ReentrantReadWriteLock();
		 
		public static void main(String[] args)  {
			final Test test = new Test();
			 
			new Thread(){
				public void run() {
					test.get(Thread.currentThread());
				};
			}.start();
			 
			new Thread(){
				public void run() {
					test.get(Thread.currentThread());
				};
			}.start();
			 
		}  
		 
		public void get(Thread thread) {
			rwl.readLock().lock();
			try {
				long start = System.currentTimeMillis();
				 
				while(System.currentTimeMillis() - start <= 1) {
					System.out.println(thread.getName()+"正在进行读操作");
				}
				System.out.println(thread.getName()+"读操作完毕");
			} finally {
				rwl.readLock().unlock();
			}
		}
	}  
	
 　　此时打印的结果为：
 
	Thread-0正在进行读操作
	Thread-0正在进行读操作
	Thread-1正在进行读操作
	Thread-0正在进行读操作
	Thread-1正在进行读操作
	Thread-0正在进行读操作
	Thread-1正在进行读操作
	Thread-1正在进行读操作
	Thread-1正在进行读操作
	Thread-1正在进行读操作
	Thread-1正在进行读操作
	Thread-1正在进行读操作
	Thread-0正在进行读操作
	Thread-0正在进行读操作
	Thread-0正在进行读操作
	Thread-0正在进行读操作
	Thread-1正在进行读操作
	Thread-1正在进行读操作
	Thread-1正在进行读操作
	Thread-1正在进行读操作
	Thread-0正在进行读操作
	Thread-1正在进行读操作
	Thread-1正在进行读操作
	Thread-0正在进行读操作
	Thread-1正在进行读操作
	Thread-1正在进行读操作
	Thread-0正在进行读操作
	Thread-1正在进行读操作
	Thread-1正在进行读操作
	Thread-1正在进行读操作
	Thread-0正在进行读操作
	Thread-1正在进行读操作
	Thread-1正在进行读操作
	Thread-0正在进行读操作
	Thread-1正在进行读操作
	Thread-0正在进行读操作
	Thread-1正在进行读操作
	Thread-0正在进行读操作
	Thread-1正在进行读操作
	Thread-0正在进行读操作
	Thread-1正在进行读操作
	Thread-0正在进行读操作
	Thread-1正在进行读操作
	Thread-0正在进行读操作
	Thread-1正在进行读操作
	Thread-0正在进行读操作
	Thread-1正在进行读操作
	Thread-0读操作完毕
	Thread-1读操作完毕

　　说明thread1和thread2在同时进行读操作。

　　这样就大大提升了读操作的效率。

　　不过要注意的是，如果有一个线程已经占用了读锁，则此时其他线程如果要申请写锁，则申请写锁的线程会一直等待释放读锁。

　　如果有一个线程已经占用了写锁，则此时其他线程如果申请写锁或者读锁，则申请的线程会一直等待释放写锁。

　　关于ReentrantReadWriteLock类中的其他方法感兴趣的朋友可以自行查阅API文档。

　　5.Lock和synchronized的选择

　　总结来说，Lock和synchronized有以下几点不同：

　　1）Lock是一个接口，而synchronized是Java中的关键字，synchronized是内置的语言实现；

　　2）synchronized在发生异常时，会自动释放线程占有的锁，因此不会导致死锁现象发生；而Lock在发生异常时，如果没有主动通过unLock()去释放锁，则很可能造成死锁现象，因此使用Lock时需要在finally块中释放锁；

　　3）Lock可以让等待锁的线程响应中断，而synchronized却不行，使用synchronized时，等待的线程会一直等待下去，不能够响应中断；

　　4）通过Lock可以知道有没有成功获取锁，而synchronized却无法办到。

　　5）Lock可以提高多个线程进行读操作的效率。

　　在性能上来说，如果竞争资源不激烈，两者的性能是差不多的，而当竞争资源非常激烈时（即有大量线程同时竞争），此时Lock的性能要远远优于synchronized。所以说，在具体使用时要根据适当情况选择。

## 三.锁的相关概念介绍

　　在前面介绍了Lock的基本使用，这一节来介绍一下与锁相关的几个概念。

　　1.可重入锁

　　如果锁具备可重入性，则称作为可重入锁。像synchronized和ReentrantLock都是可重入锁，可重入性在我看来实际上表明了锁的分配机制：基于线程的分配，而不是基于方法调用的分配。举个简单的例子，当一个线程执行到某个synchronized方法时，比如说method1，而在method1中会调用另外一个synchronized方法method2，此时线程不必重新去申请锁，而是可以直接执行方法method2。

　　看下面这段代码就明白了： 

	class MyClass {
		public synchronized void method1() {
			method2();
		}
		 
		public synchronized void method2() {
			 
		}
	}
 　　上述代码中的两个方法method1和method2都用synchronized修饰了，假如某一时刻，线程A执行到了method1，此时线程A获取了这个对象的锁，而由于method2也是synchronized方法，假如synchronized不具备可重入性，此时线程A需要重新申请锁。但是这就会造成一个问题，因为线程A已经持有了该对象的锁，而又在申请获取该对象的锁，这样就会线程A一直等待永远不会获取到的锁。

　　而由于synchronized和Lock都具备可重入性，所以不会发生上述现象。

　　2.可中断锁

　　可中断锁：顾名思义，就是可以相应中断的锁。

　　在Java中，synchronized就不是可中断锁，而Lock是可中断锁。

　　如果某一线程A正在执行锁中的代码，另一线程B正在等待获取该锁，可能由于等待时间过长，线程B不想等待了，想先处理其他事情，我们可以让它中断自己或者在别的线程中中断它，这种就是可中断锁。

　　在前面演示lockInterruptibly()的用法时已经体现了Lock的可中断性。

　　3.公平锁

　　公平锁即尽量以请求锁的顺序来获取锁。比如同是有多个线程在等待一个锁，当这个锁被释放时，等待时间最久的线程（最先请求的线程）会获得该所，这种就是公平锁。

　　非公平锁即无法保证锁的获取是按照请求锁的顺序进行的。这样就可能导致某个或者一些线程永远获取不到锁。

　　在Java中，synchronized就是非公平锁，它无法保证等待的线程获取锁的顺序。

　　而对于ReentrantLock和ReentrantReadWriteLock，它默认情况下是非公平锁，但是可以设置为公平锁。

　　看一下这2个类的源代码就清楚了：  
![ReentrantLockSource](http://op7wplti1.bkt.clouddn.com/201642145495232.jpg)
　　

　　在ReentrantLock中定义了2个静态内部类，一个是NotFairSync，一个是FairSync，分别用来实现非公平锁和公平锁。

　　我们可以在创建ReentrantLock对象时，通过以下方式来设置锁的公平性：

	ReentrantLock lock = new ReentrantLock(true);
 　　如果参数为true表示为公平锁，为fasle为非公平锁。默认情况下，如果使用无参构造器，则是非公平锁。

 ![ReentrantLockConst](http://op7wplti1.bkt.clouddn.com/201646038317744.jpg)
　　

　　另外在ReentrantLock类中定义了很多方法，比如：

　　isFair()        //判断锁是否是公平锁

　　isLocked()    //判断锁是否被任何线程获取了

　　isHeldByCurrentThread()   //判断锁是否被当前线程获取了

　　hasQueuedThreads()   //判断是否有线程在等待该锁

　　在ReentrantReadWriteLock中也有类似的方法，同样也可以设置为公平锁和非公平锁。不过要记住，ReentrantReadWriteLock并未实现Lock接口，它实现的是ReadWriteLock接口。

　　4.读写锁

　　读写锁将对一个资源（比如文件）的访问分成了2个锁，一个读锁和一个写锁。

　　正因为有了读写锁，才使得多个线程之间的读操作不会发生冲突。

　　ReadWriteLock就是读写锁，它是一个接口，ReentrantReadWriteLock实现了这个接口。

　　可以通过readLock()获取读锁，通过writeLock()获取写锁。

　　上面已经演示过了读写锁的使用方法，在此不再赘述。


参考资料：

　　http://blog.csdn.net/ns_code/article/details/17487337

　　http://houlinyan.iteye.com/blog/1112535

　　http://ifeve.com/locks/

　　http://ifeve.com/read-write-locks/

　　http://blog.csdn.net/fancyerii/article/details/6783224

　　http://blog.csdn.net/ghsau/article/details/7461369/

　　http://blog.csdn.net/zhaozhenzuo/article/details/37109015
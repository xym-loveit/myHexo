---
title: Java并发编程:如何创建线程
date: 2017-04-30 15:47:34
categories: Java并发编程系列
tags: [java多线程]
description:  在Java中，一个应用程序对应着一个JVM实例（也有地方称为JVM进程），一般来说名字默认为java.exe或者javaw.exe（windows下可以通过任务管理器查看）。Java采用的是单线程编程模型，即在我们自己的程序中如果没有主动创建线程的话，只会创建一个线程，通常称为主线程。
---

<!-- more -->

在前面一篇文章中已经讲述了在进程和线程的由来，今天就来讲一下在Java中如何创建线程，让线程去执行一个子任务。下面先讲述一下Java中的应用程序和进程相关的概念知识，然后再阐述如何创建线程以及如何创建进程。下面是本文的目录大纲：

　　一.Java中关于应用程序和进程相关的概念

　　二.Java中如何创建线程

　　三.Java中如何创建进程

　　若有不正之处，请多多谅解并欢迎批评指正。

　　请尊重作者劳动成果，转载请标明原文链接：

 　　http://www.cnblogs.com/dolphin0520/p/3913517.html

## 一.Java中关于应用程序和进程相关的概念

　　在Java中，一个应用程序对应着一个JVM实例（也有地方称为JVM进程），一般来说名字默认为java.exe或者javaw.exe（windows下可以通过任务管理器查看）。Java采用的是单线程编程模型，即在我们自己的程序中如果没有主动创建线程的话，只会创建一个线程，通常称为主线程。但是要注意，虽然只有一个线程来执行任务，不代表JVM中只有一个线程，JVM实例在创建的时候，同时会创建很多其他的线程（比如垃圾收集器线程）。

　　由于Java采用的是单线程编程模型，因此在进行UI编程时要注意将耗时的操作放在子线程中进行，以避免阻塞主线程（在UI编程时，主线程即UI线程，用来处理用户的交互事件）。

## 二.Java中如何创建线程

　　在java中如果要创建线程的话，一般有两种方式：1）继承Thread类；2）实现Runnable接口。

　　1.继承Thread类

　　继承Thread类的话，必须重写run方法，在run方法中定义需要执行的任务。

	class MyThread extends Thread{
	    private static int num = 0;
	     
	    public MyThread(){
	        num++;
	    }
	     
	    @Override
	    public void run() {
	        System.out.println("主动创建的第"+num+"个线程");
	    }
	}
 　　创建好了自己的线程类之后，就可以创建线程对象了，然后通过start()方法去启动线程。注意，不是调用run()方法启动线程，run方法中只是定义需要执行的任务，如果调用run方法，即相当于在主线程中执行run方法，跟普通的方法调用没有任何区别，此时并不会创建一个新的线程来执行定义的任务。

	public class Test {
	    public static void main(String[] args)  {
	        MyThread thread = new MyThread();
	        thread.start();
	    }
	}
 
 
	class MyThread extends Thread{
	    private static int num = 0;
	     
	    public MyThread(){
	        num++;
	    }
	     
	    @Override
	    public void run() {
	        System.out.println("主动创建的第"+num+"个线程");
	    }
	}
 　　在上面代码中，通过调用start()方法，就会创建一个新的线程了。为了分清start()方法调用和run()方法调用的区别，请看下面一个例子：

	public class Test {
	    public static void main(String[] args)  {
	        System.out.println("主线程ID:"+Thread.currentThread().getId());
	        MyThread thread1 = new MyThread("thread1");
	        thread1.start();
	        MyThread thread2 = new MyThread("thread2");
	        thread2.run();
	    }
	}
 
 
	class MyThread extends Thread{
	    private String name;
	     
	    public MyThread(String name){
	        this.name = name;
	    }
	     
	    @Override
	    public void run() {
	        System.out.println("name:"+name+" 子线程ID:"+Thread.currentThread().getId());
	    }
	}

 　　运行结果：

![线程调用运行结果](http://op7wplti1.bkt.clouddn.com/151004139364653.jpg)


　　从输出结果可以得出以下结论：

　　1）thread1和thread2的线程ID不同，thread2和主线程ID相同，说明通过run方法调用并不会创建新的线程，而是在主线程中直接运行run方法，跟普通的方法调用没有任何区别；

　　2）虽然thread1的start方法调用在thread2的run方法前面调用，但是先输出的是thread2的run方法调用的相关信息，说明新线程创建的过程不会阻塞主线程的后续执行。

　　2.实现Runnable接口

　　在Java中创建线程除了继承Thread类之外，还可以通过实现Runnable接口来实现类似的功能。实现Runnable接口必须重写其run方法。

下面是一个例子：

	public class Test {
	    public static void main(String[] args)  {
	        System.out.println("主线程ID："+Thread.currentThread().getId());
	        MyRunnable runnable = new MyRunnable();
	        Thread thread = new Thread(runnable);
	        thread.start();
	    }
	}
	 
	 
	class MyRunnable implements Runnable{
	     
	    public MyRunnable() {
	         
	    }
	     
	    @Override
	    public void run() {
	        System.out.println("子线程ID："+Thread.currentThread().getId());
	    }
	}

 　　Runnable的中文意思是“任务”，顾名思义，通过实现Runnable接口，我们定义了一个子任务，然后将子任务交由Thread去执行。注意，这种方式必须将Runnable作为Thread类的参数，然后通过Thread的start方法来创建一个新线程来执行该子任务。如果调用Runnable的run方法的话，是不会创建新线程的，这根普通的方法调用没有任何区别。

　　事实上，查看Thread类的实现源代码会发现Thread类是实现了Runnable接口的。

　　在Java中，这2种方式都可以用来创建线程去执行子任务，具体选择哪一种方式要看自己的需求。直接继承Thread类的话，可能比实现Runnable接口看起来更加简洁，但是由于Java只允许单继承，所以如果自定义类需要继承其他类，则只能选择实现Runnable接口。

## 三.Java中如何创建进程

 　　在Java中，可以通过两种方式来创建进程，总共涉及到5个主要的类。

　　第一种方式是通过Runtime.exec()方法来创建一个进程，第二种方法是通过ProcessBuilder的start方法来创建进程。下面就来讲一讲这2种方式的区别和联系。

　　首先要讲的是Process类，Process类是一个抽象类，在它里面主要有几个抽象的方法，这个可以通过查看Process类的源代码得知：

　　位于java.lang.Process路径下：

	public abstract class Process
	{
	    
	    abstract public OutputStream getOutputStream();   //获取进程的输出流
	      
	    abstract public InputStream getInputStream();    //获取进程的输入流
	 
	    abstract public InputStream getErrorStream();   //获取进程的错误流
	 
	    abstract public int waitFor() throws InterruptedException;   //让进程等待
	  
	    abstract public int exitValue();   //获取进程的退出标志
	 
	    abstract public void destroy();   //摧毁进程
	}
　　1）通过ProcessBuilder创建进程

　　ProcessBuilder是一个final类，它有两个构造器：

	public final class ProcessBuilder
	{
	    private List<String> command;
	    private File directory;
	    private Map<String,String> environment;
	    private boolean redirectErrorStream;
	 
	    public ProcessBuilder(List<String> command) {
	    if (command == null)
	        throw new NullPointerException();
	    this.command = command;
	    }
	 
	    public ProcessBuilder(String... command) {
	    this.command = new ArrayList<String>(command.length);
	    for (String arg : command)
	        this.command.add(arg);
	    }
	...
	}
 　　构造器中传递的是需要创建的进程的命令参数，第一个构造器是将命令参数放进List当中传进去，第二构造器是以不定长字符串的形式传进去。

　　那么我们接着往下看，前面提到是通过ProcessBuilder的start方法来创建一个新进程的，我们看一下start方法中具体做了哪些事情。下面是start方法的具体实现源代码：

	public Process start() throws IOException {
	// Must convert to array first -- a malicious user-supplied
	// list might try to circumvent the security check.
	String[] cmdarray = command.toArray(new String[command.size()]);
	for (String arg : cmdarray)
	    if (arg == null)
	    throw new NullPointerException();
	// Throws IndexOutOfBoundsException if command is empty
	String prog = cmdarray[0];
	 
	SecurityManager security = System.getSecurityManager();
	if (security != null)
	    security.checkExec(prog);
	 
	String dir = directory == null ? null : directory.toString();
	 
	try {
	    return ProcessImpl.start(cmdarray,
	                 environment,
	                 dir,
	                 redirectErrorStream);
	} catch (IOException e) {
	    // It's much easier for us to create a high-quality error
	    // message than the low-level C code which found the problem.
	    throw new IOException(
	    "Cannot run program \"" + prog + "\""
	    + (dir == null ? "" : " (in directory \"" + dir + "\")")
	    + ": " + e.getMessage(),
	    e);
	}
	}

 　　该方法返回一个Process对象，该方法的前面部分相当于是根据命令参数以及设置的工作目录进行一些参数设定，最重要的是try语句块里面的一句：

	return ProcessImpl.start(cmdarray,
	                    environment,
	                    dir,
	                    redirectErrorStream);
 　　说明真正创建进程的是这一句，注意调用的是ProcessImpl类的start方法，此处可以知道start必然是一个静态方法。那么ProcessImpl又是什么类呢？该类同样位于java.lang.ProcessImpl路径下，看一下该类的具体实现：

　　ProcessImpl也是一个final类，它继承了Process类：
	final class ProcessImpl extends Process {
	 
	    // System-dependent portion of ProcessBuilder.start()
	    static Process start(String cmdarray[],
	             java.util.Map<String,String> environment,
	             String dir,
	             boolean redirectErrorStream)
	    throws IOException
	    {
	    String envblock = ProcessEnvironment.toEnvironmentBlock(environment);
	    return new ProcessImpl(cmdarray, envblock, dir, redirectErrorStream);
	    }
	 ....
	}

 　　这是ProcessImpl类的start方法的具体实现，而事实上start方法中是通过这句来创建一个ProcessImpl对象的：

1
return new ProcessImpl(cmdarray, envblock, dir, redirectErrorStream);
 　　而在ProcessImpl中对Process类中的几个抽象方法进行了具体实现。

　　说明事实上通过ProcessBuilder的start方法创建的是一个ProcessImpl对象。

　　下面看一下具体使用ProcessBuilder创建进程的例子，比如我要通过ProcessBuilder来启动一个进程打开cmd，并获取ip地址信息，那么可以这么写：
	public class Test {
	    public static void main(String[] args) throws IOException  {
	        ProcessBuilder pb = new ProcessBuilder("cmd","/c","ipconfig/all");
	        Process process = pb.start();
	        Scanner scanner = new Scanner(process.getInputStream());
	         
	        while(scanner.hasNextLine()){
	            System.out.println(scanner.nextLine());
	        }
	        scanner.close();
	    }
	}
 　　第一步是最关键的，就是将命令字符串传给ProcessBuilder的构造器，一般来说，是把字符串中的每个独立的命令作为一个单独的参数，不过也可以按照顺序放入List中传进去。

　　至于其他很多具体的用法不在此进行赘述，比如通过ProcessBuilder的environment方法和directory(File directory)设置进程的环境变量以及工作目录等，感兴趣的朋友可以查看相关API文档。

　　2）通过Runtime的exec方法来创建进程

　　首先还是来看一下Runtime类和exec方法的具体实现，Runtime，顾名思义，即运行时，表示当前进程所在的虚拟机实例。

　　由于任何进程只会运行于一个虚拟机实例当中，所以在Runtime中采用了单例模式，即只会产生一个虚拟机实例：
	public class Runtime {
	    private static Runtime currentRuntime = new Runtime();
	 
	    /**
	     * Returns the runtime object associated with the current Java application.
	     * Most of the methods of class <code>Runtime</code> are instance
	     * methods and must be invoked with respect to the current runtime object.
	     *
	     * @return  the <code>Runtime</code> object associated with the current
	     *          Java application.
	     */
	    public static Runtime getRuntime() {
	    return currentRuntime;
	    }
	 
	    /** Don't let anyone else instantiate this class */
	    private Runtime() {}
	    ...
	 }
 　　从这里可以看出，由于Runtime类的构造器是private的，所以只有通过getRuntime去获取Runtime的实例。接下来着重看一下exec方法 实现，在Runtime中有多个exec的不同重载实现，但真正最后执行的是这个版本的exec方法：

	public Process exec(String[] cmdarray, String[] envp, File dir)
	   throws IOException {
	   return new ProcessBuilder(cmdarray)
	       .environment(envp)
	       .directory(dir)
	       .start();
	   }
 　　可以发现，事实上通过Runtime类的exec创建进程的话，最终还是通过ProcessBuilder类的start方法来创建的。

　　下面看一个例子，看一下通过Runtime的exec如何创建进程，还是前面的例子，调用cmd，获取ip地址信息：

	public class Test {
	    public static void main(String[] args) throws IOException  {
	        String cmd = "cmd "+"/c "+"ipconfig/all";
	        Process process = Runtime.getRuntime().exec(cmd);
	        Scanner scanner = new Scanner(process.getInputStream());
	         
	        while(scanner.hasNextLine()){
	            System.out.println(scanner.nextLine());
	        }
	        scanner.close();
	    }

	}
 　　要注意的是，exec方法不支持不定长参数（ProcessBuilder是支持不定长参数的），所以必须先把命令参数拼接好再传进去。

　　关于在Java中如何创建线程和进程的话，暂时就讲这么多了，感兴趣的朋友可以参考相关资料、

　　
> 参考资料：
>
>http://luckykapok918.blog.163.com/blog/static/205865043201210272168556/
>
>http://www.cnblogs.com/ChrisWang/archive/2009/12/02/use-java-lang-process->
>and-processbuilder-to-create-native-application-process.html
>
>http://lavasoft.blog.51cto.com/62575/15662/
>
>《Java编程思想》

# LeaningThread
学习自用
通常情况下，我们应该使用第一种方式来代替Thread.stop方法。然而以下几种方式应该使用Thread.interrupt方法来中断线程（该方法通常也会结合第一种方法使用）。
一开始使用interrupt方法时，会有莫名奇妙的感觉：难道该方法有问题？
API文档上说，该方法用于"Interrupts this thread"。请看下面的例子：
package com.polaris.thread; 
 
public class TestThread implements Runnable{ 
 
    boolean stop = false; 
    public static void main(String[] args) throws Exception { 
        Thread thread = new Thread(new TestThread(),"My Thread"); 
        System.out.println( "Starting thread..." ); 
        thread.start(); 
        Thread.sleep( 3000 ); 
        System.out.println( "Interrupting thread..." ); 
        thread.interrupt(); 
        System.out.println("线程是否中断：" + thread.isInterrupted()); 
        Thread.sleep( 3000 ); 
        System.out.println("Stopping application..." ); 
    } 
    public void run() { 
        while(!stop){ 
            System.out.println( "My Thread is running..." ); 
            // 让该循环持续一段时间，使上面的话打印次数少点 
            long time = System.currentTimeMillis(); 
            while((System.currentTimeMillis()-time < 1000)) { 
            } 
        } 
        System.out.println("My Thread exiting under request..." ); 
    } 
} 
运行后的结果是：
Starting thread...
My Thread is running...
My Thread is running...
My Thread is running...
My Thread is running...
Interrupting thread...
线程是否中断：true
My Thread is running...
My Thread is running...
My Thread is running...
Stopping application...
My Thread is running...
My Thread is running...
……
应用程序并不会退出，启动的线程没有因为调用interrupt而终止，可是从调用isInterrupted方法返回的结果可以清楚地知道该线程已经中断了。那位什么会出现这种情况呢？到底是interrupt方法出问题了还是isInterrupted方法出问题了？在Thread类中还有一个测试中断状态的方法（静态的）interrupted，换用这个方法测试，得到的结果是一样的。由此似乎应该是interrupt方法出问题了。于是，在网上有一篇文章：《 Java Thread.interrupt 害人！ 中断JAVA线程》，它详细的说明了应该如何使用interrupt来中断一个线程的执行。
实际上，在JAVA API文档中对该方法进行了详细的说明。该方法实际上只是设置了一个中断状态，当该线程由于下列原因而受阻时，这个中断状态就起作用了：
（1）如果线程在调用 Object 类的 wait()、wait(long) 或 wait(long, int) 方法，或者该类的 join()、join(long)、join(long, int)、sleep(long) 或 sleep(long, int) 方法过程中受阻，则其中断状态将被清除，它还将收到一个InterruptedException异常。这个时候，我们可以通过捕获InterruptedException异常来终止线程的执行，具体可以通过return等退出或改变共享变量的值使其退出。
（2）如果该线程在可中断的通道上的 I/O 操作中受阻，则该通道将被关闭，该线程的中断状态将被设置并且该线程将收到一个 ClosedByInterruptException。这时候处理方法一样，只是捕获的异常不一样而已。
 
其实对于这些情况有一个通用的处理方法：
package com.polaris.thread; 
 
public class TestThread2 implements Runnable{ 
 
    boolean stop = false; 
    public static void main(String[] args) throws Exception { 
        Thread thread = new Thread(new TestThread2(),"My Thread2"); 
        System.out.println( "Starting thread..." ); 
        thread.start(); 
        Thread.sleep( 3000 ); 
        System.out.println( "Interrupting thread..." ); 
        thread.interrupt(); 
        System.out.println("线程是否中断：" + thread.isInterrupted()); 
        Thread.sleep( 3000 ); 
        System.out.println("Stopping application..." ); 
    } 
    public void run() { 
        while(!stop){ 
            System.out.println( "My Thread is running..." ); 
            // 让该循环持续一段时间，使上面的话打印次数少点 
            long time = System.currentTimeMillis(); 
            while((System.currentTimeMillis()-time < 1000)) { 
            } 
            if(Thread.currentThread().isInterrupted()) { 
                return; 
            } 
        } 
        System.out.println("My Thread exiting under request..." ); 
    } 
} 
因为调用interrupt方法后，会设置线程的中断状态，所以，通过监视该状态来达到终止线程的目的。
 
总结：程序应该对线程中断作出恰当的响应。响应方式通常有三种：（来自温绍锦（昵称：温少）：http//www.cnblogs.com/jobs/）

注意：interrupted与isInterrupted方法的区别（见API文档）

当外部线程对某线程调用了thread.interrupt()方法后，java语言的处理机制如下：	

如果该线程处在可中断状态下，（调用了xx.wait()，或者Selector.select(),Thread.sleep()等特定会发生阻塞的api），那么该线程会立即被唤醒，同时会受到一个InterruptedException，同时，如果是阻塞在io上，对应的资源会被关闭。如果该线程接下来不执行“Thread.interrupted()方法（不是interrupt），那么该线程处理任何io资源的时候，都会导致这些资源关闭。当然，解决的办法就是调用一下interrupted()，不过这里需要程序员自行根据代码的逻辑来设定，根据自己的需求确认是否可以直接忽略该中断，还是应该马上退出。

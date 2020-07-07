---
title: "Java 多线程开发"
catalog: true
date: 2020-04-05 22:51:24
subtitle: ""
header-img: "/img/default.png"
tags:
- 后端
- Java
catagories:
---

### 线程和进程
在Java语言里面最大的特点就是支持多线程开发（也是位数不多支持多线程开发的语言），所以在整个Java开发学习里面多线程开发非常重要。

#### 进程
在传统的DOS系统的时代，其本身有一个特征：如果你的电脑出现了病毒，那么所有的程序都将无法执行，因为传统的DOS系统采用的是单进程的处理模式，单个时间内只允许一个程序在执行。 
后来Windows的时代，就开启了多进程的设计，于是就表示在一个时间段上可以同时运行多个程序的设计，多个程序轮流抢占系统资源，但在一个时间点上只会有单个系统在执行。而后来到了多核CPU，由于可以处理的CPU多了，那么即便有再多的进程出现，也比单核CPU处理的速度有所提升。

#### 线程
线程是在进程基础之上更小的程序单元，线程实在进程的基础之上创建并使用的，所以线程依赖于进程的支持。但是线程的启动速度比线程快很多，所以当使用多线程进行并发处理的时候，其执行的效率高于进程。

### 如何创建线程
如果想要在Java之中实现多线程的定义，那么就需要有一个专门的线程的主体类进行线程的定义。这个定义是有要求的，需要继承Thread类或实现Runnable的接口或callable接口。

如下为通过继承方式来创建线程的Demo：

```java
public class Test {

    public static void main(String[] args) {
        // 注意：启动线程时要调用start方法，而不要直接调用run方法。
        // 每一个线程只能调用一次start方法，多次调用会报异常。
        new MyThread("1").start();
        new MyThread("2").start();
        new MyThread("3").start();
    }

}

class MyThread extends Thread {

    private String title;
    public MyThread(String title) {
        this.title = title;
    }

    // 所有要执行的功能都应该在run()方法中进行定义。
    // 正常情况下所有方法实例化后都可以调用其方法，但run方法不可以直接调用。直接调用不会多线程处理
    @Override
    public void run() {
        for (int i = 0; i < 100; i++) {
            System.out.println("线程" + title + " : " + i);
        }
    }

}
```

如下为通过实现Runnable方式来创建线程的Demo，实现Runnable的方式可以避免Java单继承的局限，推荐使用这种实现方式。

```java
public class Test {

    public static void main(String[] args) {
        // 由于MyThread不再继承Thread，因此无法直接调用start方法。可以通过Thread的构造器传入实现Runnable接口的类来间接调用。
        new Thread(new MyThread("1")).start();
        new Thread(new MyThread("2")).start();
        new Thread(new MyThread("3")).start();
    }
}

/**
 * 由于Java有单继承的限制，因此通过实现Runnable接口来定义线程内容是首选。
 */
class MyThread implements Runnable {

    private String title;
    public MyThread(String title) {
        this.title = title;
    }

    @Override
    public void run() {
        for (int i = 0; i < 100; i++) {
            System.out.println("线程" + title + " : " + i);
        }
    }
}
```

下图为MLKJ的一张讲解图，讲的非常深刻，Java程序代码通过不同平台的JVM来调用不同操作系统的底层函数，进而控制CPU，内存等系统资源。
JVM 提供了start0这个接口来启动线程，屏蔽掉不同OS上的不同实现。而启动多线程调用start方法其实会去调用JVM的start0方法。
因此可以记住，没有start方法就没办法启动多线程。

![/img/thread1.png](/img/thread1.png)

### Runnable，Thread，MyThread的关系
首先Thread extends Object implements Runnable, 而我们自定义的MyThread也要实现Runnable的接口并复写run方法。
从下面的图上我们可以快速意识到，Thread其实就是利用代理模式来调度系统资源。

Thread类这个代理类中有start及其他操作系统资源的方法，我们不需要关注其复杂的底层实现，我们需要在yTread run方法中去实现的核心业务就好了。
这个设计实在是太精妙了！

![/img/thread2.png](/img/thread2.png)

### Callable
#### 如何使用Callable接口？
在JDK 1.5之后新增了Callable接口，同样可以实现多线程。实现起来优点麻烦。

![/img/thread4.png](/img/thread4.png)

如下为Callable的一个demo
```java
import java.util.concurrent.Callable;
import java.util.concurrent.ExecutionException;
import java.util.concurrent.FutureTask;

public class Test {

    public static void main(String[] args) throws ExecutionException, InterruptedException {
        FutureTask<String> futureTask = new FutureTask(new MyThread());
        new Thread(futureTask).start();
        System.out.println(futureTask.get());
    }

}

class MyThread implements Callable<String> {

    @Override
    public String call() throws Exception {
        for (int i = 0; i < 100; i++) {
            System.out.println(i);
        }
        return "程序执行完成";
    }
}
```

#### Runnable和Callable的区别
Runnable是JDK1.0就出现的实现多线程的方式，而callable是在JDK1.5之后出现的。
java.lang.Runnable接口之中只有一个run方法，并且没有返回值；java.util.concurrent.Callable 接口提供有call方法，可以提供返回值。

### 线程的生命周期
![/img/thread5.png](/img/thread5.png)
线程的生命周期包含5个阶段，包括：新建、就绪、运行、阻塞、销毁。
+ 新建：就是刚使用new方法，new出来的线程；
+ 就绪：就是调用的线程的start()方法后，这时候线程处于等待CPU分配资源阶段，谁先抢的CPU资源，谁开始执行;
+ 运行：当就绪的线程被调度并获得CPU资源时，便进入运行状态，run方法定义了线程的操作和功能;
+ 阻塞：在运行状态的时候，可能因为某些原因导致运行状态的线程变成了阻塞状态，比如sleep()、wait()之后线程就处于了阻塞状态，这个时候需要其他机制将处于阻塞状态的线程唤醒，比如调用notify或者notifyAll()方法。唤醒的线程不会立刻执行run方法，它们要再次等待CPU分配资源进入运行状态;
+ 销毁：如果线程正常执行完毕后或线程被提前强制性的终止或出现异常导致结束，那么线程就要被销毁，释放资源;
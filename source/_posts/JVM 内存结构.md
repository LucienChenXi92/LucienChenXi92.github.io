---
title: "JVM 内存结构"
catalog: true
date: 2020-03-20 22:51:24
subtitle: ""
header-img: "/img/default.png"
tags:
- 后端
- Java
catagories:
---

### 栈(Stacks)
+ 一个线程对应一个栈，线程销毁，栈空间释放。
+ 线程每调用一个方法，则有一个栈帧(Stack Frame)入栈，栈帧用于存储局部变量表，操作表，动态连接，方法出口等信息。方法执行完成，则栈帧出栈。
+ 一个栈被塞1亿个帧，空间不够时会报StackOverflow 错误。
+ 栈过多，超过栈总空间则报OOM。

### 本地方法栈(Native method stacks)
+ 本地方法栈与JVM栈所发挥的作用非常相似，只不过调用的静态方法，静态方法为C语言实现的。
+ 本地方法栈同样会有SOF和OOM两种错误

### 程序计数器(Program Counter Register)
+ 线程私有，用于记录线程运行的位置。

### 堆(Heap)
+ 虚拟机内存中最大的一块，线程共享区域，用来存放对象实例。
+ 分为新生代(young generation)和老年代(old generation)，其比例为1:2
+ 新生代，占总空间1/3，内包含：
    + 伊甸区(Eden), 占80%空间
    + 生还者0区(From Survivor)，占10%空间
    + 生还者1区(To Survivor)，占10%空间
+ 老年代，占总空间的2/3
+ 永久代：JDK 1.8后hotspot取消了永久代的概念。

### 方法区(Method area)
+ 线程共享的内存区域，用于存储已经被虚拟机加载的类信息，常量，静态变量，即时编译后的代码数据。
+ 方法区包含运行时常量池(Runtime Constant Pool)。
+ 在1.7版本之前，hotspot利用永久代来实现方法区，Java 1.8后被移至元空间。

### 元空间(Metaspace)
+ JDK 1.8后出现，为系统内存，用来代替永久代来存储方法区内容，默认情况下，Metaspace空间只受本地内存空间限制。

### 直接内存(Direct Memory)
+ 这一部分不属于JVM，但是经常会被调用，这部分空间大小可以通过JVM参数进行调整。
内存图
下面这个图画的很精髓啦，
![/img/jmm.png](/img/jmm.png)
---
title: "浅谈Javascript中对象以及原型链"
catalog: true
date: 2020-01-08 20:51:24
subtitle: ""
header-img: "/img/default.png"
tags:
- 前端
- Javascript
catagories:
---
>
> Javascript是一种非常灵活的面向对象的语言，Java虽然也是面向对象的语言，但两者差异巨大。深入学习之后才发现Javascript真的做到了万物皆对象，非常有趣。本文就类比Java语言，看看Javascript神奇之处。
>

### 类(class)
+ 在Java语言中有"类"这个概念，类是一系列对象的抽象，包括属性和行为。对象则为类的一个具体实例。相同类的不同的对象可能在属性值上会有差异，但是不同对象的属性和方法都是相同的，不会有哪个对象多出一条属性来。  
+ Javascript中是没有"类"这个语法概念的，虽然保留`class`这个关键字，但含义不一样的。

### 构造函数(constructor)
+ 在Java中，任何实现类都必须要有构造函数(构造器), 否则Java无法实例化对象。当类中没有声明任何构造函数时，JVM会在编译时帮忙创建无参构造函数。 当我们需要一个对象时，通过`new`命令调用构造函数来获得一个对象。
```java

public class Test() {
    public static void main(String[] args) {
        
        // 调用
        Person person = new Person();
    
    }
}

class Person {

    // 类的属性
    String name;
    int age;
    
    // 无参构造方法, 构造方法无返回类型，方法名和类名相同
    public Person() {}
    
    // 类的方法
    public void eat()
    
}


```

+ Javascript虽然没有类这个语法概念，但是构造函数却是有的，并且和Java中的用法相近。如果说Java中类是对象的模板，那在Javascript中，构造函数则是创建对象的模板。对象所具有的属性是在构造函数中进行声明的。创建对象时同样是`new`命令调用构造函数。但接下来这句话高能了哈，构造函数本身也是一个对象，你可以在console试一下，构造函数同样是可以点出属性的。现在不理解不要紧，往后看。
```javascript

    function Person() {
        /*
         1. 构造函数函数名开头第一个字母大写，目的是为了能够和普通函数区分开。
         2. 函数体内使用this关键字，代表了所要生成的对象实例。
          */
        this.name = "Lucien";
        this.age = 1;
    }
    
    let person = new Person();

    console.log("name: " + person.name + ", age: " + person.age)

```

### Javascript new命令的原理
    
Javascript中new命令执行步骤如下：
   1. 创建一个空对象，作为将要返回的对象实例。
   2. 将这个空对象的原型，指向构造函数的prototype属性。
   3. 将这个空对象赋值给函数内部的this关键字。
   4. 开始执行构造函数内部的代码。
   

当我第一次读到这个过程时非常不理解第二步是什么意思，查了很多资料才明白。试想一个问题，在Java中我们可以在类中声明方法，也就是所有此类对象都应该共有的方法。  

在Javascript中我们知道可以在构造函数中声明属性，那如何声明对象共有的方法呢？所有新创建的对象的原型都会指向构造函数的`prototype`属性，这句话的意思不正是告诉我们构造函数的`prototype`属性是所有对象的都可以访问的共有属性么？

所以我们可以通过如下操作来将所有person对象共有的方法声明在构造函数的`prototype`下：

```javascript
    Person.prototype.eat = function() {
        console.log('eat...')
    }
```

### 原型链

我们刚刚有在前面提到构造函数也是对象，那构造函数的构造函数又是谁呢？在Javascript中有两大内置对象，Function和Object，他们即是对象也是构造函数，Function的构造函数是Function自己，Object的构造函数也是Function。好，我们总结出Function是所有构造函数的构造函数。

既然构造函数是对象，按照这个尿性试想一下，其实一般函数也是对象，属性也是对象，Javascript中万物都是对象！

刚刚讲new的操作中的第二步，对象的原型指向其构造器的`prototype`属性，我们可以理解为这个属性其实也是个对象，是对象就会有原型，再指向更上层的原型。直到最高层的对象Obejct。现在你可以看到，一条原型的链路已经显示出来了，这个就是原型链。


### 原型链有什么用？
我们可以简单的记，如果我们调用某个对象的某个方法或属性，如果在当前对象下查不到的话，Javascript会去其原型对象，甚至原型对象的原型下面查找，直到找到为止。如果最终找到所有对象的源头Object下面仍没有找到，则会通过Object.`__proto__` = null跳出查找。


掘金上有一篇文章讲的非常透测，读完之后清晰了不少，把link附在这里：  
[用自己的方式（图）理解constructor、prototype、__proto__和原型链](https://juejin.im/post/5cc99fdfe51d453b440236c3)



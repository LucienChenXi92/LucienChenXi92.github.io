---
title: "Javascript中公共属性及方法的两种声明方式对比"
catalog: true
date: 2020-05-11 12:51:24
subtitle: "解析特权属性和原型属性对比"
header-img: "/img/default.png"
tags:
- 前端
- Javascript
catagories:
---

>
> 之前写过一篇日志[《浅谈Javascript中对象以及原型链》](/2020/01/08/浅谈Javascript中对象以及原型链)，在这篇日志中有介绍过如何声明对象公有的属性和方法，今天我们来深入聊一下Prototype声明和在构造器中声明的两种方式的异同。
> 

在解释这两种声明方法之前，我们需要回忆一下Javascript 通过`new`指令创建对象的过程：
>   1. 创建一个空对象，作为将要返回的对象实例。
>   2. 将这个空对象的原型，指向构造函数的prototype属性。
>   3. 将这个空对象赋值给函数内部的this关键字。
>   4. 开始执行构造函数内部的代码。

### 构造函数中声明
构造函数中可以声明共有的对象属性，同时也可以声明共有的方法，这种声明的属性和方法又称为`特权属性`和`特权方法`。我们之前有了解过通过构造函数创建对象的过程，第三步中 `新创建的空对象会被赋值给构造函数内部的this关键字`，换句话说，所声明的属性和方法是直接挂在新创建的对象下面的。

我们来举个例子：
```javascript
function Person() {
    this.age = 0;   // 声明Number类型属性
    this.eat = function() {     // 声明共有方法
        console.log("吃东西。。。")
    }
}
```

### 在原型中声明
我们接着刚才介绍过的Javascript构造过程来讲，其第二步 `将这个空对象的原型，指向构造函数的prototype属性。` 中讲到所有新创建的对象的原型都会指向他们共有的构造函数的prototype属性上，因此prototype下所声明的属性和方法即可被所有同类对象访问到。我们把这种方式声明的属性和方法称为`原型属性`和`原型方法`。
```javascript
Person.prototype = {
    age: 10,
    name: "Lucien",
    drink: function() {
        console.log("喝东西。。。")
    },
    eat: function() {
        console.log("被喂东西。。。")
    }
}
```

### 两种声明方式的差异
我们先来写个测试：
```javascript
var person = new Person;

console.log("age: " + person.age)   // age: 0
console.log("name: " + person.name)     // name: Lucien

person.drink();     // 喝东西。。。
person.eat();       // 吃东西。。。
```
通过以上结果我们发现：
1. 无论采用哪种声明方式，都可以让新创建对象上带上已声明的属性或方法。
2. 当某一属性同时在构造器中和原型中声明时，`特权属性`值会覆盖`原型属性`的值，方法也是同理。也就是说`特权属性`有更高的优先级。

### 为什么特权属性更高呢？
在Javascript中，当访问一个对象的属性或方法时，Javascript会从对象本身开始往上遍历整个原型链，直到找到对应属性和方法为止。如果此时到达了原型链的最顶端，既`Object.prototype`仍未能找到时返回undefined。从此我们明白了，`特权属性`由于是直接挂在新建对象上的，所以它更早被Javascript捕获，而`原型属性` 是挂在其上一层对象的prototype上的，所以更晚被捕获。我们可以通过Debug来印证这一点，在person对象下，只有`特权属性` age 和`特权方法` eat，而`原型方法`和`原型属性` 则是在`person.__proto__` 下面。
![/img/javascript_constructor.png](/img/javascript_constructor.png)

---
title: "ES6语法"
catalog: true
date: 2020-05-24 16:51:24
subtitle: "React学习系列第一篇"
header-img: "/img/default.png"
tags:
- 前端
- React
catagories:
---

### 什么是ES6
**ECMAScript 6** 是在2015年6月发布的新的ECMAScript标准，因此也被称为**ES6**，或**ECMAScript 2015**。在2018年底时，各大浏览器厂商都实现了对ES6语法的支持。具体版本和时间如下：

| 浏览器 | 发布时间 |
| ---- | ---- |
| Chrome 58 | Jan 2017 |
| Edge 14 | Aug 2016 |
| Firefox 54 | Mar 2017 |
| Safari 10 | Jul 2016 |
| Opera 55 | Aug 2018 |

### ES6新特性有哪些
#### 箭头函数和关键字this
ES6中的箭头函数和JDK1.8中的箭头函数非常相似，即支持表达式形式，也支持表达体形式。另外与函数不同，箭头函数与其周围的代码共享相同的`this`变量(this指向当前对象本身)。如果箭头函数在另一个函数内，则它共享其父函数的`arguments`变量。
```Javascript
// Expression bodies
var evens = [0,2,4,6,8,10];
var odds = evens.map(v => v + 1);   // [1,3,5,7,9,11]
var nums = evens.map((v, i) => v + i);  // [0,3,6,9,12,15]

// Statement bodies
var fives = [];     
nums.forEach(v => {
  if (v % 5 === 0)
    fives.push(v);
});
// fives = [0,15]

// Lexical this
var bob = {
    _name: "Bob",
    _friends: ["Lucien", "Fish"],
    printFriends() {
      this._friends.forEach(f =>
        console.log(this._name + " knows " + f));   // this指向当前对象本身,
    }
};
bob.printFriends();     

// Lexical arguments
function square() {    
  let example = () => {
    let numbers = [];
    for (let number of arguments) {     // 注意，箭头函数体内中并未声明变量arguments，此变量是从外部square函数中继承来的。
      numbers.push(number * number);
    }
    return numbers;
  };
  return example();
}

square(2, 4, 7.5, 8, 11.5, 21); // returns: [4, 16, 56.25, 64, 132.25, 441]
```

#### 类(Classes)
ES6中的Classes是一种面向对象编程的语法糖，通过一种简单便捷的声明方式使得类模式更加方便使用。类也支持基于原型链的继承，父类调用(super call)，实例方法和静态方法，还有构造器等等。这个功能在React开发中经常被用到，每次创建组件时都要继承React.Component，目的是为了获得React.Component中生命周期函数。
```Javascript
class Person {
    constructor() {     // 构造器函数
        this.name = "人"; // 实例属性
        this.age = 0;   // 实例属性
        this.eat = function() {     // 实例方法
            console.log("吃东西。。。")
        };
    }

    drink() {   // 实例方法，这种声明方式和在构造其中声明函数效果都是一致的，都会挂载在构造器的prototype属性下面。
        console.log("喝东西。。。")
    }

    static yell() {   // 静态方法
        console.log("啊，啊，啊！！！！")
    }
}

class Son extends Person {
    drink() {   // 覆写父类方法
        console.log("喝饮料。。。")
    }
}

const son = new Son;  // 创建实例
console.log(son.age);   // 0
console.log(son.name);  // “人”
console.log(son.height);    // undefined
son.drink();    // 喝饮料。。。
son.eat();      // 吃东西。。。
Person.yell();  // 啊，啊，啊！！！！
Son.yell();     // 啊，啊，啊！！！！
```

#### 对象的拓展(Enhanced Object Literals)
对象的拓展使得开发时可以直接在构造函数中设置对象的原型，方法和父类调用(super call)。这些拓展使得基于对象属性和类的声明更加紧密，对象设计更加便捷。
```Javascript
var obj = {
    // Sets the prototype. "__proto__" or '__proto__' would also work.
    __proto__: theProtoObj,
    // Computed property name does not set prototype or trigger early error for
    // duplicate __proto__ properties.
    ['__proto__']: somethingElse,
    // Shorthand for ‘handler: handler’
    handler,
    // Methods
    toString() {
     // Super calls
     return "d " + super.toString();
    },
    // Computed (dynamic) property names
    [ "prop_" + (() => 42)() ]: 42
};
```

#### 字符串模版(Template Strings)
字符串模版提供了构造字符串的语法糖，类似于Perl，Python中字符串插值功能。可选则地，可以添加标签以允许对字符串结构进行自定义，从而避免注入攻击或从字符串内容构建更高级别的数据结构。
```Javascript
// Basic literal string creation
const sampleString = `This is a pretty little template string.`;

// Multiline strings
const multilineString = `In ES5 this is 
 not legal.`

// Interpolate variable bindings
var name = "Bob", time = "today";
const bindingString = `Hello ${name}, how are you ${time}?`

// Unescaped template strings
const unescapedString = String.raw`In ES5 "\n" is a line-feed.`;// String.raw保证转义字符不会生效

// Construct an complated json body
const foo = '{a: 1, b: {c: 2}}'
const bar = 'abc'
const requestBody = `{foo: ${foo}, bar: ${bar}}`

console.log("sampleString: " + sampleString);   
// sampleString: This is a pretty little template string.
console.log("multilineString: " + multilineString); 
// multilineString: In ES5 this is 
// not legal.
console.log("bindingString: " + bindingString);
// bindingString: Hello Bob, how are you today?
console.log("unescapedString: " + unescapedString);
// unescapedString: In ES5 "\n" is a line-feed.
console.log("requestBody: " + requestBody)
// requestBody: {foo: {a: 1, b: {c: 2}}, bar: abc}
```

#### 解构(Destructuring)
解构是ES6新特性中非常重要且实用的一个新功能，它允许通过匹配模式进行进行获取数据的value，并且支持匹配数组和对象。解构是fail-soft的，匹配不到不会报错，而是返回`undefined`作为结果。在React中我们经常会将props中的变量提出来，如 `const {path, url}  = this.props`，这样就可以避免写太多次的this.props。
```Javascript
// list matching
var [a, ,b] = [1,2,3];
console.log('a === 1: ' + a === 1);
b === 3;
console.log('b === 3: ' + b === 3);

// define a object first
var goods = {
    cars: {
        color: "red"
    },
    foods: ["water", "bread", "milk"],
    tickets: {
        movie: {
            name: "Harry Potter"
        }
    }
}

// object matching
var { cars: color, foods, tickets: {movie: {name: x}} } = goods
console.log("color: " + color["color"]);
console.log("foods[0]: " + foods[0]);
console.log("tickets.movie.name: " + x);

// object matching shorthand
var {cars, foods, tickets} = goods

// Can be used in parameter position
function g({name: x}) {
  console.log("g({name: 5}): " + x);
}
g({name: 5})

// Fail-soft destructuring
var [a] = [];
console.log(`a === undefined: ` + (a === undefined));

// Fail-soft destructuring with defaults
var [a = 1] = [];
console.log('a === 1: ' + (a === 1));

// Destructuring + defaults arguments
function r({x, y, w = 10, h = 10}) {
  return x + y + w + h;
}
console.log('r({x:1, y:2}) === 23: ' + (r({x:1, y:2}) === 23));
```
#### Default + Rest + Spread
Default是指允许在声明函数时设置默认参数。
```Javascript Default
function f(x, y=12) {
  // y is 12 if not passed (or passed as undefined)
  return x + y;
}
console.log(`f(3) == 15: ` + (f(3) == 15))
```
Rest是指将函数中多个参数合并成一个数组。
```Javascript Rest
function f(x, ...y) {   // 非常类似Java中的可变参数，将多个参数合并成一个数组y
    return x + ": " + y.join(',');
}
console.log(f("内容","a","b","c"));
```
拓展运算符Spread是Rest参数的逆运算。
```Javascript Spread
function f(array,items) {   // 等同于写一个pushAll函数，将数组中的元素依次传入push方法中
    return array.push(...items);
}
const array = ["d"];
const items = [ "a", "b", "c"];
f(array,items)
console.log(array)  // 【“d”,"a","b","c"】
```

#### Let 和 Const
let和const是ES6新的声明变量的方式，let用于声明变量，const则用于声明常量。let和const都没有变量提升的特性。目前在React项目中都会使用const和let去声明变量。
```Javascript
function f() {
  {
    let x;
    {
      // this is ok since it's a block scoped name
      const x = "sneaky";
      // error, was just defined with `const` above
      x = "foo";
    }
    // this is ok since it was declared with `let`
    x = "bar";
    // error, already declared above in this block
    let x = "inner";
  }
}
```

#### 迭代器和For...Of
迭代器（Iterator）是一个接口，为各种不同的数据结构提供统一的访问机制。任何数据只要部署了 Iterator 接口，就可以完成遍历操作。 `for...of`方法会调用迭代器对象下Symbol.iterator方法。迭代器的本身是一个对象，这个对象有 next() 方法返回结果对象，这个结果对象有下一个返回值 value、迭代完成布尔值 done。
如下示例为用迭代器来遍历斐波那契数列。
```Javascript
let fibonacci = {
    [Symbol.iterator]() {
      let pre = 0, cur = 1;
      return {
        next() {
          [pre, cur] = [cur, pre + cur];
          return { done: false, value: cur }
        }
      }
    }
  }
  
  for (var n of fibonacci) {
    // truncate the sequence at 1000
    if (n > 1000)
      break;
    console.log(n);
  }
```

#### 模块(Modules)
对组件定义模块的语言级别支持。整理来自流行的JavaScript模块加载器（AMD，CommonJS）的模式。
```Javascript
// lib/math.js
export function sum(x, y) {
  return x + y;
}
export var pi = 3.141593;

// app.js
import * as math from "lib/math";
console.log("2π = " + math.sum(math.pi, math.pi));

// otherApp.js
import {sum, pi} from "lib/math";
console.log("2π = " + sum(pi, pi));

// lib/mathplusplus.js
export * from "lib/math";
export var e = 2.71828182846;
export default function(x) {
    return Math.exp(x);
}

// app.js
import exp, {pi, e} from "lib/mathplusplus";
console.log("e^π = " + exp(pi));
```

#### Set 和 Map
更高效的数据解构，和Java中的用法一致。
```Javascript
// Sets
var s = new Set();
s.add("hello").add("goodbye").add("hello");
s.size === 2;
s.has("hello") === true;

// Maps
var m = new Map();
m.set("hello", 42);
m.set(s, 34);
m.get(s) == 34;
```
#### Math, Number, String, Object APIs 
```Javascript 
Number.EPSILON
Number.isInteger(Infinity) // false
Number.isNaN("NaN") // false

Math.acosh(3) // 1.762747174039086
Math.hypot(3, 4) // 5
Math.imul(Math.pow(2, 32) - 1, Math.pow(2, 32) - 2) // 2

"abcde".includes("cd") // true
"abc".repeat(3) // "abcabcabc"

Array.from(document.querySelectorAll("*")) // Returns a real Array
Array.of(1, 2, 3) // Similar to new Array(...), but without special one-arg behavior
[0, 0, 0].fill(7, 1) // [0,7,7]
[1,2,3].findIndex(x => x == 2) // 1
["a", "b", "c"].entries() // iterator [0, "a"], [1,"b"], [2,"c"]
["a", "b", "c"].keys() // iterator 0, 1, 2
["a", "b", "c"].values() // iterator "a", "b", "c"

Object.assign(Point, { origin: new Point(0,0) })
```

#### Promises
Promises 是一个异步编程的库，目前已广泛应用于各个网络通讯的库中，如fetch，axios等等。
```Javascript
function timeout(duration = 0) {
    return new Promise((resolve, reject) => {
        setTimeout(resolve, duration);
    })
}

var p = timeout(1000).then(() => {
    return timeout(2000);
}).then(() => {
    throw new Error("hmm");
}).catch(err => {
    return Promise.all([timeout(100), timeout(200)]);
})
```
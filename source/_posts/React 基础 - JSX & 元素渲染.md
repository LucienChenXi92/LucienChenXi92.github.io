---
title: "React 基础 - JSX & 元素渲染"
catalog: true
date: 2020-07-20 10:51:24
subtitle: "React学习系列第二篇"
header-img: "/img/default.png"
tags:
- 前端
- React
catagories:
---

学习React最好的起点就是[React官网](https://zh-hans.reactjs.org/docs/getting-started.html)，个人觉得React官网对中文的支持度还是蛮高的。Follow React官网的示例，第一个值得我们学习的知识点就是JSX。

## JSX(Javascript XML)

刚接触React的时候最大惊喜就是可以把Js代码和Html标签写在一起，非常的灵活。深入了解了一下，原来JSX仅仅是React.createElement(component, props, ...children)函数的语法糖。如下JSX代码：
```javascript
<MyButton color="blue" shadowSize={2}>
    Click Me
</MyButton>
```
最终编译为：
```javascript
React.createElement(
    MyButton,
    {color: 'blue', shadowSize: 2}, 
    'Click Me'
)
```

### 使用JSX的前提准备

#### babel

首先你要明白JSX是Javascript内定义的一套XML语法，可以编译成Javascript代码。而浏览器是无法识别的未编译处理的JSX代码的，因此想要在浏览器中运行，必须要进行预编译处理。React官方给出的解决方案是`babel`。最简单的方式是通过`<script>`标签直接引入到项目中：
```javascript
<script src="https://unpkg.com/babel-standalone@6/babel.min.js"></script>
```
然后就可以在任何`<script>`标签内使用JSX语法了，方法是为其添加`type="text/babel"`属性。
如果你的项目是React项目并且已经安装了Node.js，可以通过`npm install babel-cli@6 babel-preset-react-app@3`来安装。

#### React必须在作用域内

由于JSX会编译为React.createElement调用形式，所以React库也必须包含在JSX代码作用域内。
例如，在如下代码中，虽然React和CustomButton并没有被直接使用，但还是需要导入：
```javascript
import React from 'react';
import CustomButton from './CustomButton';

function WarningButton() {
  // return React.createElement(CustomButton, {color: 'red'}, null);
  return <CustomButton color="red" />;
}
```
如果你不使用JavaScript打包工具而是直接通过 `<script>` 标签加载 React，则必须将React挂载到全局变量中。

### JSX用法

#### 在JSX中嵌入表达式

在下面的例子中，我们声明了一个名为`name`的变量，然后在JSX中使用它，并将它包裹在大括号中：
```javascript
const name = 'Josh Perez';
const element = <h1>Hello, {name}</h1>;

ReactDOM.render(
  element,
  document.getElementById('root')
);
```
在JSX语法中，你可以在`大括号`内放置任何有效的JavaScript表达式。例如，`2 + 2`，`user.firstName` 或 `formatName(user)` 都是有效的 JavaScript表达式。

在下面的示例中，我们将调用JavaScript函数formatName(user)的结果，并将结果嵌入到`<h1>`元素中。
```javascript
function formatName(user) {
  return user.firstName + ' ' + user.lastName;
}

const user = {
  firstName: 'Harper',
  lastName: 'Perez'
};

const element = (
  <h1>
    Hello, {formatName(user)}!
  </h1>
);

ReactDOM.render(
  element,
  document.getElementById('root')
);
```

#### JSX也是一个表达式

在编译之后，JSX表达式会被转为普通JavaScript函数调用，并且对其取值后得到JavaScript对象。

也就是说，你可以在`if`语句和`for`循环的代码块中使用JSX，将JSX赋值给变量，把JSX当作参数传入，以及从函数中返回JSX：
```html
function getGreeting(user) {
  if (user) {
    return <h1>Hello, {formatName(user)}!</h1>;
  }
  return <h1>Hello, Stranger.</h1>;
}
```

#### JSX特定属性

可以使用大括号，来在属性值中插入一个JavaScript表达式：
```javascript
const element = <img src={user.avatarUrl}></img>;
```

#### JSX指定子元素

JSX 标签里能够包含很多子元素:
```javascript
const element = (
  <div>
    <h1>Hello!</h1>
    <h2>Good to see you here.</h2>
  </div>
);
```

## 元素渲染

元素是构成React应用的最小砖块。与浏览器的DOM元素不同，React元素是创建开销极小的普通对象，`React DOM`会负责更新DOM来与React元素保持一致。
> `注意： 元素不等于组件，元素是更小的单位。`

### 渲染元素为DOM

想要将React元素须髯到DOM节点，只需要将他们一起传入ReactDOM.render():
```javascript
<div id="root"></div>
const element = <h1>Hello, world</h1>;
ReactDOM.render(element, document.getElementById('root'));
```
### 更新已渲染的元素

> `React元素是不可变对象。一旦被创建，你就无法更改它的子元素或者属性。一个元素就像电影的单帧：它代表了某个特定时刻的UI。`

根据我们已有的知识，更新UI唯一的方式是创建一个全新的元素，并将其传入 ReactDOM.render()。考虑一个计时器的例子：
```javascript
function tick() {
  const element = (
    <div>
      <h1>Hello, world!</h1>
      <h2>It is {new Date().toLocaleTimeString()}.</h2>
    </div>
  );
  ReactDOM.render(element, document.getElementById('root'));
}
setInterval(tick, 1000);
```

### React只更新需要更新的部分

React DOM会将元素和它的子元素与它们之前的状态进行比较，并只会进行必要的更新来使DOM达到预期的状态。`也正是因为这个特性，React的性能优于传统的Jquery性能。`

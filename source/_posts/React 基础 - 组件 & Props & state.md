---
title: "React 基础 - 组件 & Props & state"
catalog: true
date: 2020-07-20 11:51:24
subtitle: "React学习系列第三篇"
header-img: "/img/default.png"
tags:
- 前端
- React
catagories:
---

## 组件

组件，从概念上类似于JavaScript函数。它接受任意的入参（即 “props”），并返回用于描述页面展示内容的React元素。

### 函数组件与class组件

#### 函数组件

```javascript
function Welcome(props) {
  return <h1>Hello, {props.name}</h1>;
}
```
该函数是一个有效的React组件，因为它接收唯一带有数据的 “props”（代表属性）对象与并返回一个React元素。这类组件被称为“函数组件”，因为它本质上就是JavaScript函数。

#### class组件

通过以下五步将 Clock 的函数组件转成 class 组件：

1. 创建一个同名的ES6 class，并且继承于React.Component。
2. 添加一个空的render()方法。
3. 将函数体移动到render()方法之中。
4. 在render()方法中使用this.props替换props。
5. 删除剩余的空函数声明。

```javascript
class Welcome extends React.Component {
  render() {
    return <h1>Hello, {this.props.name}</h1>;
  }
}
```

#### state & props

state和props是React中两个重要的变量，一般情况下其中任何一个变量发生变化都会导致组件重新渲染。

+ state 可以看作是组件自身的一些属性，调用时用`this.state`来获取，通过`this.setState()`函数来进行更新，`切不可用this.state.state1 = newValue`方式更新state，不会生效。
+ props 可以看作是父组件赋予给子组件的属性，其值不可改变。

> 有时有些变量在多个组件中都有应用到，一般可以选择从更高层组件往下传递，但这种共享变量的方式有两个弊端，一是当组件层级过深的时候，增加了项目组件之间的耦合度。二是当各子组件都需要能够更新共享变量时，不便于变量的管理和控制。
> 因此，当出现如上两种情况时，更加推荐适用`Redux`等状态管理工具。

```javascript
class Clock extends React.Component {
  constructor(props) {
    super(props);
    this.state = {date: new Date()};
  }

  render() {
    return (
      <div>
        <h1>Hello, world!</h1>
        <h2>It is {this.state.date.toLocaleTimeString()}.</h2>
      </div>
    );
  }
}
```
#### 数据向下流动

不管是父组件或是子组件都无法知道某个组件是有状态的还是无状态的，并且它们也并不关心它是函数组件还是class组件。

这就是为什么称state为局部的或是封装的的原因。除了拥有并设置了它的组件，其他组件都无法访问。

组件可以选择把它的state作为props向下传递到它的子组件中：

```javascript
<FormattedDate date={this.state.date} />
```
这通常会被叫做“自上而下”或是“单向”的数据流。任何的state总是所属于特定的组件，而且从该state派生的任何数据或UI只能影响树中“低于”它们的组件。

如果你把一个以组件构成的树想象成一个props的数据瀑布的话，那么每一个组件的state就像是在任意一点上给瀑布增加额外的水源，但是它只能向下流动。


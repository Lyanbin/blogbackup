---
title: React 101
date: 2017-03-28 16:34:40
tags: [JavaScript, React]
---


临走前坑一把，体验一把React。本篇即为新手入门的101。

<!-- more -->

## 一、快速开始

以下为通常情况下的组件创建
```javascript
import React from 'react';
import ReactDOM from 'react-dom';
// 注意组件开头第一个字母都要大写
class App extends React.Component {
// 若是需要绑定 this. 方法或是需要在 constructor 使用 props ，定义 state ，就需要 constructor 。若是在其他方法（如 render）使用 this.props 则不用一定要定义 constructor


    constructor(props) {
// extends 可以继承 React.Component 。super 方法可以呼叫继承父类构造函数。
        super(props);
// 初始化 state，等于 ES5 中的 getInitialState
        this.state = {

        };
    }
// render 是 Class based 组件唯一必须的方法（method）
// 每个组件，只能有一个顶级标签！！！！
    render() {
        return (
            <div>
                <h1>Hello, World!</h1>
            </div>
        );
    }
}

// 将 <App /> 组件插入 id 为 app 的 DOM 元素中
ReactDOM.render(<App />, document.getElementById('app'));
```

当创建的组件仅仅为无状态组件时，可以使用剪头函数快捷创建
```javascript

//
const MyComponent = () => (
	<div>Hello, World!</div>
);

// 将 <MyComponent /> 组件插入 id 为 app 的 DOM 元素中
ReactDOM.render(<MyComponent />, document.getElementById('app'));
```

为了提高组件的健壮性，通常会对传入的props进行验证。
```javascript

// PropTypes 验证，若传入的 props type 不符合将会显示错误
App.propTypes = {
    todo: React.PropTypes.object,
    name: React.PropTypes.string,
}

// Prop 预设值，若对应 props 没传入值将会使用 default 值
App.defaultProps = {
    todo: {},
    name: '',
}
```

## 二、props和state

props为父元素向下传递数据的方式

```javascript
// 这里输出 Hello，lyb
const MyComponent = () => (
	<div>Hello, {this.props.name}!</div>
);
ReactDOM.render(<MyComponent name="lyb"/>, document.getElementById('app'));
```
state为当前组件的状态，可以通过setState({name: 'lyb01'})这样的方式来更新组件内容
以下代码，会在按钮点击后，将lyb改变为lyb01

```javascript
class App extends React.Component {
    constructor(props) {
        super(props);
        this.state = {
        	name: 'lyb'
        }
        // 在构造函数中，需要将自有的方法绑定到当前的this中去
        this.change = this.change.bind(this);
    }
    change () {
        // change后，组件本身会重新执行render进行渲染
        this.setState({
            name: 'lyb01'
        });
    }
    // jsx可以嵌入js表达式，但是需要用花括号包起来。需要注意的是，js的表达式为变量名、函数定义表达式 、属性访问表达式、函数调用表达式、算数表达式、关系表达式、逻辑表达式。但是，if和for循环这类不是js表达式，不可以直接写入jsx
    render () {
        return (
            <div>
                {this.state.name}
                {/* jsx的注释，需要同样需要使用花括号包起来 */}
                <button onClick={this.change}>change</button>
	        </div>
        );
    }
}

ReactDOM.render(
	<App />, document.getElementById('app')
);
```


## 三、生命周期钩子

React每个组件，都具有三个状态，分别为
* Mount 挂载组件
* Update 正在被重新渲染
* Unmount 销毁组件

```javascript
componentDidMount () {
    // 该函数会在组件挂载成功后调用
}

ComponentWillUnmount () {
    // 该函数会在组件销毁前进行执行
}


```

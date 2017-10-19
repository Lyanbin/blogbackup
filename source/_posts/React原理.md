---
title: React原理
date: 2017-10-19 17:37:55
tags: [JavaScript, React]
---


### React是声明式的
在React中，写一个组件，通常会这么写
```js
class Mycomponent extends React.Component {
    render () {
        return <div>hello</div>
    }
}
```
其中，return的内容，会被编译为
```js
class Mycomponent extends React.Component {
    render () {
        return React.createElement('div', null, 'hello');
    }
}
```
在某种意义上说，我们是通过调用了React.createElement来创建
一个组件。但是，事实上我们仅仅是声明了这个组件，然后React帮助我们实例化了这个组件对象，并调用render()来创建了这个组件。我们仅仅需要描述我们需要什么，剩下的全部由React渲染页面。


<!-- more -->

### React的渲染基础渲染原理
接下来，靠着这个思想，来创建一个假的React，这里暂且叫它为Feact吧。

首先，要实现React，我们分解问题来看，就是要实现一个

```js
Feact.render(<h1>hello world</h1>, document.getElementById('root'));

```
简单了解React就会知道，render()方法的第一个参数，是jsx语法，这里暂时不考虑jsx的编译，问题继续被降级，那么就是要实现一个

```js
Feact.render(
    Feact.createElement('h1', null, 'hello world'),
    document.getElementById('root')
);

```
也就是说，Feact对象中，目前必须得有2个方法，一个是createElement(), 一个是render()。其中，createElement()方法返回一个普通的json对象，来描述dom。所以，目前，我们的Feact对象应该张这个样子

```js
const Feact = {
   createElement(type, props, children) {
       const element = {
           type,
           props: props || {}
       };
       
       if (children) {
            element.props.children = children;
       }
       
       return element;
   }
   render() {
    
   }
}

```

那么render函数，从上面可以得知，render()接收两个参数，一个是描述dom的对象，一个是被插入的位置。也就是说render()应该长的像这样子

```js
render(element, container) {
    const componentInstance = new FeactDomComponent(element);
    return componentInstance.mountComponent(container);
}
```
其中，FeactDomComponent对象用来生成dom，然后调用该对象中的mountComponent方法来实现挂载。所以FeactDomComponent对象至少应该是这个样子

```js
class FeactDomComponent {
    /* element 即为描述dom的json对象 */
    constructor(element) {
        this._currentElement = element;
    }
    
    mountComponent(container) {
        const domElement = document.createElement(this._currentElement.type);
        const text = this._currentElement.props.text;
        const textNode = document.createTextNode(text);
        domElement.appendChild(textNode);
        
        container.appendChild(domElement);
        
        return domElement;
    }
    
}
```

给以上代码组装下，一个最基本的Feact就组件完成了。

### 用户自定义组件

以上得到的仅仅是一个写死的组件，下面加入用户可配置功能。所以，Feact需要添加一个createClass()方法。

```js
const Feact = {

    createElement() {
        /* 跟上头一样 */
    }

    createClass(spec) {
        function Constructor(props) {
            this.props = props;
        }
        
        Constructor.prototype.render = spec.render;
        
        return Constructor;
    }
    
    render(element, container) {
        /*
            之前的render不能接收用户自定义的组件，这里待会儿修改
        */
    }
    
};

const MyTitle = Feact.createClass({
    render() {
        return Feact.createElement('h1', null, this.props.message);
    }
});

Feact.render({
   Feact.createElement(MyTitle, { message: 'i am here' }),
   document.getElementById('root')
});

```

现在，可以Feact可以接收自定义的组件了。就差改写render()方法了。但是，目前来看，render()方法利用FeactDomComponent对象仅仅处理原生的dom。需要进行改造，添加一个FeactCompositeComponentWrapper来包裹FeactDomComponent对象

```js
render(element, container) {
    const componentInstance = new FeactCompositeComponentWrapper(element);
    return componentInstance.mountComponent(container);
}

class FeactCompositeComponentWrapper {
    constructor(element) {
        this._currentElement = element;
    }
    
    mountComponent(container) {
        /* 目前Component是一个构造函数 */
        const Component = this._currentElement.type;
        
        const componentInstance = new Component(this._currentElement.props);
        
        const element = componentInstance.render();
        
        const domComponentInstance = new FeactDOMComponent(element);
        return domComponentInstance.mountComponent(container);
    }
}

```

到目前为止，Feact已经能处理一些用户自定义的组件以及原生组件，但是在render()方法中，假如存在一些业务逻辑的话，目前还不能胜任，比如一个组件如下所示：

```js
const MyMessage = Feact.createClass({
    render() {
        if (this.props.asTitle) {
            retrun Feact.createElement(Mytitle, {
                message: this.props.message
            });
        } else {
            return Feact.createElement('p', null, this.props.message)
        }
    }
});
```

这样的话，我们必须在FeactCompositeComponentWrapper中的mountComponent方法里，做一些简单的逻辑判断。那么，FeactCompositeComponentWrapper改写为：

```js
class FeactCompositeComponentWrapper {
    constructor(element) {
        this._currentELement = element;
    }
    mountComponent(container) {
        const Component = this._currentElement.type;
        const componentInstance = new Component(this._currentElement.props);
        
        let element = componentInstance.render();
        
        while (typeof element.type === 'function') {
            element = (new element.type(element.props)).render();
        }
        
        const domComponentInstance = new FeactDomComponent(element);
        domComponentInstance.mountComponent(container);
    }
}

```

目前来看，Feact已经更加完善了，但是目前的生命周期还有问题，后续再解决。
另外，在Feact.render()执行的时候，无论组件是自定义组件还是原生的dom，我们发现他们终归会执行到FeactDOMComponent，所以这里为了统一处理，我们可以给所有的组件包裹一个工厂函数，让他成为自定义组件，方便我们后续处理。

工厂函数很简单，如下所示

```js
const TopLevelWrapper = function (props) {
    this.props = props;
}

TopLevelWrapper.prototype.render = function () {
    return this.props;
}

const Feact = {
    render(element, container) {
        const warpperElement = this.createElement(TopLevelWrapper, element);
        const componentInstance = new FeactCompositeComponentWrapper(wrapperElement);
        return componentInstance.mountComponent(container);
    }
}

```
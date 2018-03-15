---
title: React原理（二）生命周期
date: 2018-03-15 17:30:55
tags: [JavaScript, React]
---


上一篇中，简单的介绍了ReactJs的渲染结构，这里，会对组件的生命周期进行简单的分析。

首先，对createClass()这个函数进行简单的处理。



上一节，createClass()如下所示：

``` javascript
const Feact = {
    
    // 其他都一样

    createClass (spec) {
       function Constructor(props) {
           this.props = props;
       }
       Constructor.prototype.render = spec.render;

       return Constructor;
    }
}
```

从上面可以看出，render()方法仅仅接收了子组件的render方法，这里可以简单处理下，将整个spec挂载到组件的原型上去。让Constructor()能够继承包括componentWillMount等更多的属性和方法。

<!-- more -->

``` javascript
const Feact = {
    
    // 其他都一样

    createClass (spec) {
       function Constructor(props) {
           this.props = props;
       }
       Constructor.prototype = Object.assign(Constructor.prototype, spec);

       return Constructor;
    }
}
```

接下来，我们开始改造FeactCompositeComponentWrapper对象。


在上一章中，我们记得，FeactCompositeComponentWrapper的mountComponent()方法走了个『捷径』，疯狂递归子组件的render方法，直到出现一个原生的dom对象为止。现在有一个问题，循环的过程中，这个『捷径』仅仅调用了子组件的render()方法，而生命周期函数却无处安放。

``` javascript
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
从上面可以看出，mountComponent逐步向下，直到找到原生的组件。只要render()返回了一个自定义组件，他就会继续调用render()，直到得到一个原生的组件。这就导致了，这些子组件，没有办法参与到整个挂载的生命周期中去。换句话说，它们的render()方法是被调用了，但是也就是仅仅被调用了而已。现在，我们需要做的事情，就是将每一个完整组件参与到整个挂载过程中。


现在，开始改造下FeactCompositeComponentWrapper对象：

``` javascript
class FeactCompositeComponentWrapper {
    constructor(element) {
        this._currentELement = element;
    }
    mountComponent(container) {
        const Component = this._currentElement.type;
        const componentInstance = new Component(this._currentElement.props);
        
        // let element = componentInstance.render();
        // while (typeof element.type === 'function') {
        //     element = (new element.type(element.props)).render();
        // }

        this._instance = componentInstance;
        const markup = this.performInitialMount(container);
        return markup;
        
        // const domComponentInstance = new FeactDomComponent(element);
        // domComponentInstance.mountComponent(container);
    }

    performInitialMount(container) {
        const renderedElement = this._instance.render();
        const child = instantiateFeactComponent(renderedElement);
        return FeactReconciler.mountComponent(child, container);        
    }
}

const FeactReconciler = {
    mountComponent(internalInstance, container) {
        return internalInstance.mountComponent(container);
    }
}

function instantiateFeactComponent(element) {
    if (typeof element.type === 'string') {
        return new FeactDOMComponent(element);
    } else if (typeof element.type === 'function') {
        return new FeactCompositeComponentWrapper(element);
    }
}

```

一下子写了介么多代码，传递了一个基本的思想，就是将mounting的过程，单独提出来。也就是FeactReconciler的作用，它将在以后承担更多的工作。在React中，也存在一个ReactReconciler，跟我们的FeactReconciler一样。

处理 Feact.render()

Feact.render()在上一章中，调用了componentInstance.mountComponent(container)，现在，讲其更新为我们的刚刚写的FeactReconciler

``` javascript
const Feact = {

    // 其他都一样

    render(element, container) {
        const warpperElement = this.createElement(TopLevelWrapper, element);
        const componentInstance = new FeactCompositeComponentWrapper(wrapperElement);
        return FeactReconciler.mountComponent(
            componentInstance,
            container
        );
    }
}

```
至此，所有的自定义组件，已经被完全挂载，这些自定义组件将参与到整个挂载的生命周期中去。

最后，增加componentWillMount和componentDidMount

接下来就简单啦，增加钩子函数即可。

``` javascript

class FeactCompositeComponentWrapper {
    
    // 其他都一样

    mountComponent(container) {
        const Component = this._currentElement.type;
        const componentInstance =  new Component(this._currentElement.props);
        this._instance = componentInstance;

        if (componentInstance.componentWillMount) {
            componentInstance.componentWillMount();
        }

        const markUp = this.performInitialMount(container);

        if (componentInstance.componentDidMount) {
            componentInstance.componentDidMount();
        }

        return markUp;
    },
    
    // 其他都一样

}

```
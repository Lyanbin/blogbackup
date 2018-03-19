---
title: React原理（三）更新
date: 2018-03-19 17:08:21
tags: [JavaScript, React]
---

上两篇中，`React`具有了基本的渲染能力。但是，一旦渲染发生，就不能再改变了。这一篇中，我们将在`render`中，添加更新功能,并且，将简单的展示`虚拟dom`的`diff`过程。


### 简单更新

让React应用实现更新，最普通的办法就是调用组件的`setState()`方法。但是，`React`也支持通过`React.render()`来实现更新。就像如下所示：

``` javascript
React.render(<h1>hello</h1>, root);

setTimeout(() => {
    React.render(<h1>hello again</h1>, root);
}, 2000)
```

本篇中，我们暂时忽略`setState()`，先通过`Feact.render()`来实现更新。说实话，这就是最屌丝的『`props`改变，所以更新』的模型，即如果你又`render`了，并且传入了不同的`props`给子组件，那么就更新呗。

### 开始

理念非常简单，`Feact.render()`只需检查，之前是否渲染过，如果渲染过，就执行`update`。结构如下所示

``` javascript

const Feact = {

    // 其他都一样

    render(element, container) {
        const prevComponent = getTopLevelComponentInContainer(container);
        if (prevComponent) {
            return updateRootComponent(
                prevComponent,
                element
            );
        } else {
            return renderNewRootComponent(element, container);
        }
    }

    // 其他都一样

}

function renderNewRootComponent(element, container) {
    // 这个函数就是之前的render内的逻辑
    const wrapperElement = Feact.createElement(TopLevelWrapper, element);
    const componentInstance = new FeactCompositeComponentWrapper(wrapperElement);

    return FeactReconciler.mountComponent(componentInstance, container);
};

function getTopLevelComponentInContainer(container) {
    // TODO
}

function updateRootComponent(prevComponent, nextElement) {
    // TODO
}

```

看起来很美好，如果之前渲染过，则将之前的组件和更新后的组件，传递给一个函数，这个函数将计算出`dom`所需要做出的更新动作。否则，就像上一篇所讲的那样，直接将组件渲染进来即可。那么，问题已经被降级为搞定我们缺失的2个函数。


### 记住我们所做的

对于每一次渲染，我们需要记录那些我们渲染过的组件，以获取他们的引用，方便后续的渲染。咋整呢？最好的方法就是在创建`dom`节点的时候，做上一个标记。

``` javascript

function renderNewRootComponent(element, container) {

    const wrapperElement = Feact.createElement(TopLevelWrapper, element);
    const componentInstance = new FeactCompositeComponentWrapper(wrapperElement);

    // 注意这里操作
    const markUp = FeactReconciler.mountComponent(componentInstance, container);

    // 多了这么一行，这里将组件的实例保存到container上
    // 这里我们想要的是组件实例的_renderedComonent, 因为componentInstance是最顶级的一个壳子，无需更新
    // 还记得_renderedComponent么？在上一节的预挂载函数内，我们偷摸的保存了下。
    container.__feactComponentInstance = componentInstance._renderedComponent;
    return markUp;
}

```

那么，对于已经挂在的组件的情况，一样的返回`container.__feactComponentInstance`。

``` javascript
function getTopLevelComponentInContainer(container) {
    return container.__feactComponentInstance;
}
```

### 更新

首先先看一个简单的示例。

``` javascript

Feact.render(
    Feact.createElement('h1', null, 'hello'),
    root
);

setTimeout(() => {
    Feact.render(
        Feact.createElement('h1', null, 'hello again'),
        root
    );
}, 2000);

```

2秒后，我们又一次调用了`Feact.render()`，但是，这次调用所传入的元素大概长这样

``` javascript
{
    type: 'h1',
    props: {
        children: 'hello again'
    }
}

```

当Feact确定了这是一个更新动作，则会进入到`updateRootComponent()`函数中，

``` javascript

function updateRootComponet(prevComponet, nextElement) {
    prevComponent.receiveComponent(nextElement);
};

```

这里注意，我们没有创建一个新的组件，`prevComponent`是我们第一次渲染的时候就创建的组件，现在只是更新了它自己而已。所以，组件一旦被创建，它将一直存在，直到被卸载（`unmount`）。

再来考虑`FeactDOMComponent`

``` javascript

class FeactDOMComponent {

    // 其他都一样

    receiveComponent(nextElement) {
        const prevElement = this._currentElement;
        this.updateComponent(prevElement, nextElement);
    }

    updateComponent(prevElement, nextElement) {
        const lastProps = prevElement.props;
        const nextProps = nextElement.props;

        this._updateDOMProperties(lastProps, nextProps);
        this._updateDOMChildren(lastProps, nextProps);

        this._currentElement = nextElement;
    };

    _updateDOMProperties(lastProps, nextProps) {
        // 更新css
    }

    _updateDOMChildren(lastProps, nextProps) {
        // 更新组件
    }
}

```

`receiveComponent()`只是调用了`updateComponent()`，而`updateComponent()`则最终调用了`_updateDOMProperties()`和`_updateDOMChildren()`，这2个函数最终，完成了真实`dom`的更新。需要注意的是，`_updateDOMProperties()`更多的关注了`CSS`相关的内容。简便期间，我们暂时不考虑它，仅仅指出，在`React`中，这个函数是用来解决样式的更新的。

`_updateDOMChildren()`在`React`中，那可是相当的复杂，主要是解决了各种不同的场景下的执行情况。但是，在`Feact`中，为了方便理解，我们只考虑子节点是文本的情况，也就是上文中所写的，我们从`hello`，更新到了`hello again`。

``` javascript
class FeactDOMComponent {

    // 其他都一样
    _updateDOMChildren(lastProps, nextProps) {
        const lastContent = lastProps.children;
        const nextContent = nextProps.children;

        if (!nextContent) {
            this.updateTextContent('');
        } else if (lastContent !== nextContent) {
            this.updateTextContent('' + nextContent);
        }
    }

    _updateTextContent(text) {
        const node = this._hostNode;

        const firstChild = node.firstChild;

        if (firstChild && firstChild === node.lastChild && firstChild.nodeType ===3) {
            firstChild.nodeValue = text;
            return;
        }

        node.textContent = text;
    }
}
```

从上面可以看出，`Feact`的`_updateDOMChildren`非常屌丝，但是大概原理就是这样。


### 更新自定义组件

上面这些内容，我们实现了`FeactDOMComponent`的更新，但是下面这种情况就无能为力了。

``` javascript

Feact.render(
    Feact.createElement(MyCoolComponent, {myProp: 'hello'}),
    document.getElementById('root');
);

setTimeout(() => {
    Feact.render(
        Feact.createElement(MyCoolComponent, {myProp: 'hello again'}),
        document.getElementById('root');
    );
}, 2000);

```

更新自定义组件就有趣多了，这也是`React`的牛逼之处。有一个好消息，自定义组件的更新，归根结底会降级到原生组件的更新，所以上面我们做的工作，都是有效的，没有浪费。

还有个更好的消息，`updateRootComponent`在执行的时候，并不关心组件是自定义的组件，还是原生的组件。他只是调用`receiveComponent`，所以，我们需要做的，只是给`FeactCompositeComponentWrapper`也增加一个`receiveComponent`就好啦。

``` javascript

class FeactCompositeComponentWrapper {

    // 其他都一样
    receiveComponent(nextElement) {
        const prevElement = this._currentElement;
        this.updateComponent(prevElement, nextElement);
    }

    updateComponent(prevComponent, nextElement) {
        const nextProps = nextElement.props;

        this._performComponentUpdate(nextElement, nextProps);
    }

    _performComponentUpdate(nextElement, nextProps) {
        this._currentELement = nextElement;
        const inst = this._instance;

        inst.props = nextProps;

        this._updateRenderedComponent();
    }

    _updateRenderedComponent() {
        const prevComponentInstance = this._renderedComponent;
        const inst = this._instance;
        const nextRenderedElement = inst.render();

        prevComponentInstance.receiveComponent(nextRenderedElement);
    }
}

```
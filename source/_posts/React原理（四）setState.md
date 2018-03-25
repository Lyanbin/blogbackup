---
title: React原理（四）setState
date: 2018-03-25 21:14:50
tags: [JavaScript, React]
---

在这一节中，我们将给`Feact`增加`setState`方法，这个方法非常有趣，端起饮料好好享受吧。

### 给`Feact`添加`state`

`state`和`props`非常相似，他们都是组件在渲染的时候，流动在内部的数据。不同的是`props`来自于外部，`state`是内部的。到目前为止，`Feact`只支持`props`，所以，在我们搞出`setState`之前，我们得先给这个小框架增加个`state`这个概念。

<!-- more -->

#### `getInitialState`

当我们挂载一个新组件的时候，我们需要给这个组件增加一个初始`state`，这个时候，我们需要调用`getInitialState`这个生命周期函数。该生命周期函数需要在实例化的时候被调用，所以，我们需要在`Feact.createClass`构造函数里增加钩子。

``` javascript
const Feact = {
    createClass(spec) {
        function Constructor(props) {
            this.props = props;

            const initialState = this.getInitialState ? this.getInitialState : null;

            this.state = initialState;
        }

        Constructor.prototype = Object.assign(Constructor.prototype, spec);

        return Constructor;
    }
}

```

就像`props`一样，我们给组件实例增加`state`、

> 注意，当我们组件没有`getInitialState`定义的时候，`state`的初始状态是`null`，`React`不会给`state`添加默认值为空对象。所以，当想使用`state`的时候，必须利用这个方法，返回一个对象。否则，如果在使用`this.state.foo`这样的操作的时候，第一次`render`会爆炸的哦。

现在，有了`getInitialState`，`Feact`组件可以随时使用`this.state`这个方法啦。

### 增加简单的`setState()`

现在，我们准备给`setState`在`Feact.createClass`中，找一个合适的位置。为了实现它，我们将给所有的通过`Feact.createClass`创建的组件一个`prototype`，这个`prototype`将拥有一个`setState`方法。

``` javascript
function FeactComponent() {

}

FeactComponent.prototype.setState = function () {
    // TODO
}

function mixSpecIntoComponent(Constructor, spec) {
    const proto = Constructor.prototype;

    for (const key in spec) {
        proto[key] = spec[key];
    }
}

const Feact = {
    createClass(spec) {
        function Constructor(props) {
            this.props = props;

            const initialState = this.getInitialState ? this.getInitialState() : null;

            this.state = initialState;
        }

        Constructor.prototype = new FeactComponent();

        mixSpecIntoComponent(Constructor, spec);

        return Constructor;
    }
}

```

`misSpecIntoComponent`在`React`中，那可是相当的复杂，当然，也更健壮，它的角色更像是`mixins`，同时，保证用户在使用的时候，不会因为这个函数翻车。

### 让`setState`代入到`updateComponent`方法中

回顾上一节，我们通过`FeactCompositeComponentWrapper.receiveComponent`来实现一个组件的更新，而这个函数接下来调用了`updateComponent`方法，所以，看起来我们只要通过`updateComponent`来处理`state`就能实现更新。那么，我们只需要将`FeactComponent.prototype.setState`和`FeactCompositeComponentWrapper.receiveComponent`打通即可。

在`React`中，有『公共实例』和『内部实例』的概念。公共实例是通过`createClass`创建的组件的实例，内部实例是`React`内部对象的实例。那么，在这些概念下，内部实例就是`FeactCompositeComponentWrapper`，不难发现，内部实例能够感知到公共实例的一切，但是反过来却不行。现在，我们准备改变这个。`setState`是公共实例给内部实例通信的方法，带着这个想法，看如下实现

``` javascript
function FeactComponent() {

}

FeactComponent.prototype.setState = function (partialState) {
    const internalInstance = getMyInternalInstancePlease(this);

    internalInstance._pendingPartialState = partialState;

    FeactReconciler.performUpdateIfNecessary(internalInstance);
}

```

`React`解决`getMyInternalInstancePlease`这个问题的方法是通过一个实例映射，这个映射保存了某个公共实例内的内部实例。

``` javascript
const FeactInstanceMap = {
    set(key, value) {
        key.__feactInternalInstance = value;
    },

    get(key) {
        return key.__feactInternalInstance;
    }
}
```

而这个映射关系的建立，是在组件挂载的时候。

``` javascript
const FeactCompositeComponentWrapper {

    // 其他都一样

    mountComponent(container) {
        const Component = this._currentElement.type;
        const componentInstance = new Component(this._currentElement.props);

        this._instance = componentInstance;

        FeactInstanceMap.set(componentInstance, this);
    }
}

```

现在，还有一个没有用到的方法，`FeactReconciler.performUpdateIfNecessary`，这个方法就像其他的协调器方法一样

``` javascript
const FeactReconciler {
    
    // 其他都一样
    performUpdateIfNecessary(internalInstance) {
        internalInstance.performUpdateIfNecessary();
    }
}

class FeactCompositeComponentWrapper {
    
    // 其他都一样
    performUpdateIfNecessary() {
        // 注意这里，一样哦
        this.updateComponent(this._currentElement, this._currentElement);
    }
}
```

最终，我们终于调用了`updateComponent`，但是请注意，这里我们做了一点HACK，虽然我们调用了更新，但是，我们传递了相同的两个参数。任何时候，当`updateComponet`传递了相同的元素，`React`就知道，只有`state`更新了，否则就是`props`更新了。`React`会通过`prevElement !== nextElement`来判断是否调用`componentWillReceiveProps`，所以，这里先改造下`Feact`，让它也做相同的处理。

``` javascript
class FeactCompositeComponentWrapper {

    // 其他都一样
    updateComponent(prevElement, nextElement) {
        const nextProps = nextElement.props;
        const inst = this._instance;

        const willReceive = prevElement !== nextElement;

        if (willReceive && inst.componentWillReceiveProps) {
            inst.componentWillReceiveProps(nextProps);
        }
        // 其他都一样
    }
}

```

这个只是`updateComponent`的片段，只是为了解决`setState()`并不会导致`componentWillReceiveProps`在渲染前的调用。也就是说，`setState`无需影响到`props`。

#### 通过新的`state`更新

现在处理`updateCompoent`，内部实例已经通过`internalInstance._pendingPartialState`获取到了新的`state`，所以现在我们需要做的，仅仅是让这个组件再渲染一次。

``` javascript
class FeactCompositeComponentWrapper {

    // 其他都一样

    updateComponent(prevElement, nextElement) {
        const nextProps = nextElement.props;
        const inst = this._instance;

        const willReceive = prevElement !== nextElement;

        if (willReceive && inst.componentWillReceiveProps) {
            inst.componentWillReceiveProps(nextProps);
        }

        let shouldUpdate = true;
        const nextState = Object.assignn({}, inst.state, this._pendingPartialState);

        this._pendingPartialState = null;

        if (inst.shouldComponentUpdate) {
            shouldUpdate = inst.shouldComponentUpdate(nextProps, nextState);
        }

        if (shouldUpdate) {
            this._performComponentUpdate(nextElement, nextProps, nextState);
        } else {
            inst.props = nextProps;
            inst.state = nextState;
        }
    }

    _performComponentUpdate(nextElement, nextProps, nextState) {
        this._currentElement = nextElement;
        const inst = this._instance;

        inst.props = nextProps;
        inst.state = nextState;

        this._updateRenderedComponent();
    }

    _updateRenderedComponent() {
        const prevComponentInstance = this._renderedComponent;
        const inst = this._instance;
        const nextRenderedElement = inst.render();

        FeactReconciler.receiveComponent(
            prevComponentInstance,
            nextRenderedElement
        );
    }
}

```

组件的更新跟之前很像，不同的是，我们增加了`state`的赋值操作。因为`state`仅仅挂载在公共实例上，`_performComponentUpdate`只改变了一行，`_updateRenderedComponent`一行没变。真正改变的重点是`updateComponent`中，我们合并`state`的操作。
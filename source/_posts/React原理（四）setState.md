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

组件的更新跟之前很像，不同的是，我们增加了`state`的赋值操作。因为`state`仅仅挂载在公共实例上，`_performComponentUpdate`只改变了一行，`_updateRenderedComponent`一行没变。真正改变的重点就是在`updateComponent`中，我们合并`state`的操作。


至此，`setState`的功能，已经基本完成啦！

但是，上面的`setState`的实现，比较屌丝，性能也比较糟糕。主要的问题是，每次调用`setState`都会导致组件的渲染。这将迫使用户，要么好好想想怎么组装数据然后只使用一次`setState`，要么就接受这种每次调用就渲染的问题。接下来，我们要做的就是改造它，使它最好能自适应的具有批量工作的能力，从而减少渲染的次数。

### 批量调用`setState`

仔细观察生命周期函数的调用，不难发现，每次的渲染，都调用了`componentWillReceiveProps`。如果用户在`componentWillReceiveProps`中调用`setState`会发生什么？在当前的代码中，这将会导致在第一次渲染过程中又一次新的渲染，而对`state`改版而造成的`props`的响应，画面太美不敢看。所以，我们最好将一系列的`state`和`props`的改变，都塞到同一次渲染中。

#### 首先我们需要给需要批量的操作保存起来

首先想到的就是改造`_pendingPartialState`，让它成为一个数组。

``` javascript
function FeactComponent() {

}
FeactComponent.prototype.setState = function (partialState) {
    const internalInstance = FeactInstanceMap.get(this);

    internalInstance._pendingPartialState = internalInstance._pendingPartialState || [];

    internalInstance._pendingPartialState.push(partialState);

    // 其他都一样
}
```

而在`updateComponent`中，调用我们将要设计的合并`state`方法。

``` javascript
class FeactCompositeComponentWrapper {
    // 其他都一样

    updateComponent(prevElement, nextElement) {
        // 其他都一样
        const nextState = this._processPendingState();
        // 其他都一样
    }

    _processPendingState() {
        const inst = this._instance;
        if (!this._pendingPartialState) {
            return inst.state;
        }

        let nextState = inst.state;

        for (let i = 0; i < this._pendingPartialState.length; ++i) {
            nextState = Object.assign(nextState, this._pendingPartialState[i]);
        }

        this._pendingPartialState = null;

        return nextState;
    }
}

```
#### 其次将批量合并之后的`state`塞到一次渲染过程中

> 注意这里的批量操作原理是非常简单的，并不是`React`中的全部功能。我们主要指出批量操作的原理。

在`Feact`中，我们只在页面还在渲染的时候，批量合并`state`，其他时候，我们并不做这样的处理。所以，在`updateComponent`过程中，我们会做一个标记，告诉外面，我们正在渲染，在渲染结束之后，讲其设置为`false`。如果`setState`看到了这个标记为`true`，他会挂起这个`state`，但不渲染它。因为它知道，当当前的渲染结束的时候，渲染引擎会重拾这个`state`进行下一次渲染。

``` javascript
class FeactCompositeComponent {

    // 其他都一样

    updateComponent(prevElement, nextElement) {
        this._rendering = true;
        // 中间这一部分跟之前一样
        this._rendering = false;
    };

}

function FeactComponent() {

}

FeactComponent.prototype.setState = function (partialState) {
    const internalInstance = FeactInstanceMap.get(this);

    internalInstance._pendingPartialState = internalInstance._pendingPartialState || [];

    internalInstance.push(partialState);

    if (!internalInstance._rendering) {
        FeactReconciler.performUpdateIfNecessary(internalInstance);
    }

}
```

基本上完成啦。


### `setState`陷阱

现在，我们已经明白了`setState`的工作原理以及批量工作的概念，但是这里有几个关于`setState`的陷阱需要注意。我们知道，当我们利用`state`去更新组件的时候，有好几个步骤，每个步骤中，被挂起的`state`需要一个一个的处理，也就是说，当我们在`setState`中使用`this.state`是非常危险的

``` javascritp
componentWillReceiveProps(nextProps) {
    this.setState({ counter: this.state.counter + 1 });
    this.setState({ counter: this.state.counter + 1 });
}
```

这个例子中，我们期待执行2次加法运算。但是，`state`会被批量处理，所以第二次`setState`和第一次的`setState`有相同的输入，所以，加法运算只会执行一次。

`React`中，解决这个问题的方法是传入一个回调函数

``` javascript
componentWillReceiveProps(nextProps) {
    this.setState((currentState) => ({
        counter: currentState.counter + 1
    });
    this.setState((currentState) => ({
        counter: currentState.counter + 1
    });
}
```

当传入回调函数的时候，我们将会得到正确的结果，我们将这个特性运用到`Feact`中去

``` javascript
_processPendingState() {
    const inst = this._instance;
    if (!this._pendingPartialState) {
        return inst.state;
    }

    let nextState = inst.state;

    for (let i = 0; i < this._pendingPartialState.length; ++i) {
        const partialState = this._pendingPartialState[i];

        if (typeof partialState === 'function') {
            nextState = partialState(nextState);
        } else {
            nextState = Object.assign(nextState, patialState);
        }
    }

    this._pendingPartialState = null;
    return nextState;
}

```

至此，大功告成啦！
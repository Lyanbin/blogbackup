---
title: Promise
date: 2019-01-28 11:29:32
tags: [JavaScript]
---

## 一、最简单Promnise

根据定义可知，`promise`是一个规范，规范并不在意`promise`是怎样`create`、`reject`，`fulfill`。规范只在意有木有一个`then`方法。
根据我们下面的使用方式
```js
let p = new MyPromise(function (resolve, reject) {
    setTimeout(() => {
        resolve(100);
    }, 5000);
});

p.then(function (data) {
    console.log(data);
});
```

<!-- more -->
可以清楚的感知到，我们传入了个函数，这个函数会在内部执行。同时，`then`方法会注册一个回调，该回调在`resolve`后进行执行，并且拿到`resolve`的传参。那么，我们的`promise`可以简单定义为：
```js
function MyPromise(executor) {

    let self = this;

    self.value = undefined;
    self.reason = undefined;

    self.onFulfilled = null;
    self.onRejected = null;

    function resolve(value) {
        self.value = value;
        self.onFulfilled(value);
    }

    function reject(reason) {
        self.reason = reason;
        self.onRejected(reason);
    }
    
    try {
        executor(resolve, reject);
    } catch (e) {
        console.log(e);
    }
}

MyPromise.prototype.then = function (onFulfilled, onRejected) {
    this.onFulfilled = onFulfilled;
    this.onRejected = onRejected;
}



let p = new MyPromise(function (resolve, reject) {
    setTimeout(() => {
        resolve(100);
    }, 5000);
});

p.then(function (data) {
    console.log(data);
});

```

## 二、不异步行不行
要知道我们正常使用`Promise`传入的函数，可以是异步函数，可以是同步函数。但是，当我们试着给`setTimeout`干掉之后，会发现报错。
```js
let p = new MyPromise(function (resolve, reject) {
    resolve(100);
});

p.then(function (data) {
    console.log(data);
});
```
具体的错误发生在上面的`resolve`定义的地方
```js
function resolve(value) {
    self.value = value;
    self.onFulfilled(value);
}
```
因为`resolve`执行的时候，`then`注册的回调函数还没有挂载在`self.onFulfilled`上。解决办法很简单，只需将`resolve`方法的主体，使用setTimeout包裹即可。`reject`同理。
则上面的最简单的`MyPromise`就变成了：

```js
function MyPromise(executor) {

    let self = this;

    self.value = undefined;
    self.reason = undefined;

    self.onFulfilled = null;
    self.onRejected = null;

    function resolve(value) {
        // 包个setTimeout
        setTimeout(() => {
            self.value = value;
            self.onFulfilled(value);
        });
    }

    function reject(reason) {
        // 包个setTimeout
        setTimeout(() => {
            self.reason = reason;
            self.onRejected(reason);
        });
    }
    
    try {
        executor(resolve, reject);
    } catch (e) {
        console.log(e);
    }
}

MyPromise.prototype.then = function (onFulfilled, onRejected) {
    this.onFulfilled = onFulfilled;
    this.onRejected = onRejected;
}

let p = new MyPromise(function (resolve, reject) {
    resolve(100);
});
p.then(function (data) {
    console.log(data);
});

```

## 三、状态
我们都知道，每个`promise`具有3个状态。`pending`、`fulfilled`、`rejected`。在`pending`状态下可以分别向其他状态转换，而且`fulfilled`、`rejected`的状态不能改变。那么，`MyPromise`可以如下表示
```js
function MyPromise(executor) {

    let self = this;

    self.value = undefined;
    self.reason = undefined;
    // 加个状态
    self.status = 'pending';
    self.onFulfilled = null;
    self.onRejected = null;

    function resolve(value) {
        if (self.status === 'pending') {
            setTimeout(() => {
                // 改变状态
                self.status = 'fulfilled';
                self.value = value;
                self.onFulfilled(value);
            });
        }
    }

    function reject(reason) {
        if (self.status === 'pending') {
            setTimeout(() => {
                // 改变状态
                self.status = 'rejected';
                self.reason = reason;
                self.onRejected(reason);
            });
        }
    }
    
    try {
        executor(resolve, reject);
    } catch (e) {
        console.log(e);
    }
}

MyPromise.prototype.then = function (onFulfilled, onRejected) {
    if (this.status === 'pending') {
        // 如果是pending，则注册
        this.onFulfilled = onFulfilled;
        this.onRejected = onRejected;
    } else if (this.status === 'fulfilled') {
        // 如果是fulfilled则直接执行
        onFulfilled();
    } else if (this.status === 'rejected') {
        // 如果是rejected则直接执行
        onRejected();
    }
}

let p = new MyPromise(function (resolve, reject) {
    resolve(100);
});
p.then(function (data) {
    console.log(data);
});
```

## 四、链式操作
目前来看，只能使用一个`then`，要想使用`promise.then(func1).then(func2).then(func3)`这种操作，还需要对回调函数的注册进行一番改造。操作起来非常简单，只需在全局注册2个数组用来存放回调函数，在`pending`时push，在`resolve`时遍历回调数组，并挨个调用即可。这里需要谨记，在函数主体（后面简称`main`）执行的时候，`then`接收的参数，理论上应该是一个函数，如果不是函数，后续我们的`MyPromise`会做一些容错处理。`main`执行到`then`，会将`then`内的参数（函数）挂起，到下一轮事件循环时候再去执行。对这里不太清楚的小伙伴可以了解下`microtask`和`macrotask`。
```js
function MyPromise(executor) {

    let self = this;

    self.value = undefined;
    self.reason = undefined;
    self.status = 'pending';
    // 变成数组
    self.onFulfilledCallbacks = [];
    self.onRejectedCallbacks = [];

    function resolve(value) {
        if (self.status === 'pending') {
            setTimeout(() => {
                self.status = 'fulfilled';
                self.value = value;
                // 遍历回调数组，并调用
                self.onFulfilledCallbacks.forEach(callback => {
                    callback(self.value);
                });
            });
        }
    }

    function reject(reason) {
        if (self.status === 'pending') {
            setTimeout(() => {
                self.status = 'rejected';
                self.reason = reason;
                // 遍历循环数组，并调用
                self.onRejectedCallbacks.forEach(callback => {
                    callback(self.value);
                });
            });
        }
    }
    
    try {
        executor(resolve, reject);
    } catch (e) {
        console.log(e);
    }
}

MyPromise.prototype.then = function (onFulfilled, onRejected) {
    if (this.status === 'pending') {
        this.onFulfilledCallbacks.push(onFulfilled);
        this.onRejectedCallbacks.push(onRejected);
    } else if (this.status === 'fulfilled') {
        onFulfilled();
    } else if (this.status === 'rejected') {
        onRejected();
    }
    // 返回自己，才能链式操作
    return this;
}

let p = new MyPromise(function (resolve, reject) {
    resolve(100);
});
p.then(function (data) {
    console.log(data);
}).then(function (data) {
    console.log(data);
});
```
上述代码看起来没啥问题，但是在诸如以下使用时，
```js
let p = new MyPromise(function (resolve, reject) {
    resolve(100);
});
p.then(function (data) {
    setTimeout(() => {
        console.log(data + 1);
    }, 100);
}).then(function (data) {
    console.log(data);
});
```
会打印100 101。也就是说，第二个`then`先执行了。而且`data`的值，由于读的是同一个`promise`实例的`data`，在`resolve`的时候，也写死了，没有办法继续透传改变。要想实现每一个`then`都读上一个`then`的`value`，我们立刻可以想到，`then`应该返回一个新的`promise`，他拥有自己的`data`，这样一级一级往下传递，就实现了真正的异步串行操作。

## 五、`then`的改造
回到上一节最后抛出的问题，上面的所有步骤，`then`中都直接返回了`this`，但是，问题来了，我们的需求为：每次调用`then`传参，取决于上一个`then`回调的返回值。看下面片段
```js
MyPromise.prototype.then = function (onFulfilled, onRejected) {
    let self = this;
    let promise2;

    if (self.status === 'fulfilled') {
        return promise2 = new MyPromise(function (resolve, reject) {

        })
    } else if (self.status === 'rejected') {
        return promise2 = new MyPromise(function (resolve, reject) {

        })
    } else if (self.status === 'pending') {
        return promise2 = new MyPromise(function (resolve, reject) {

        })
    }
}
```
可以看到，`then`中当前（对，当前，看清楚，我是说当前）的`promise`有3个状态，我们分别返回一个新的（新的，我new了，new了之后，下一级`then`内的this会变）`promise`。

另外考虑如下使用方式：
```js
let p = new MyPromise(function (resolve, reject) {
    resolve(100);
});
p.then(function (data) {
    return data + 200
}).then(function (data) {
    console.log(data);
});
```
我们希望最后的`console.log(data)`的结果是`300`，那么，在`then`的内部，`promise2`应该先拿到上一个`MyPromise`的值（也就是`onFulfilled`或者`onRejected`的值），然后马上`resolve`掉这个内部的`promise`。注意，这个`resolve`是为了传递内部的`promise`，也就是`promise2`的值，所以不管什么情况，最后只管`resolve`就是了。
```js
MyPromise.prototype.then = function (onFulfilled, onRejected) {
    let self = this;
    let promise2;

    if (self.status === 'fulfilled') {
        return promise2 = new MyPromise(function (resolve, reject) {
            try {
               let x = onFulfilled(self.data);
                resolve(x); 
            } catch (e) {
                reject(e);
            }
        });
    } else if (self.status === 'rejected') {
        return promise2 = new MyPromise(function (resolve, reject) {
            try {
                let x = onRejected(self.data);
                resolve(x);
            } catch (e) {
                reject(e)
            }
        });
    } else if (self.status === 'pending') {
        return promise2 = new MyPromise(function (resolve, reject) {
            // 如果是`pending`状态，则一样只需给当前的数组压入回调即可
            self.onFulfilledCallbacks.push((value) => {
                try {
                    let x = onFulfilled(value);
                    resolve(x);
                } catch (e) {
                    reject(e);
                }
            });
            self.onRejectedCallbacks.push((reason) => {
                try {
                    let x = onRejected(reason);
                    resolve(x);
                } catch (e) {
                    reject(e);
                }
            });
        });
    }
}
```
到目前为止，看似都很美好。但是如果`onFulfilled`返回的`x`是一个新的`MyPromise`呢？所以，在`Promise2`中，不能简单的`resolve`这个`promise2`。我们这里定义个函数，用来专门处理`x`的值。
```js
// 这里传入4个参数，x为promise2的onFulfilled或者onRejected的值，resolve和reject就是promise2的2个实参，至于为啥传递个`promise2`进来，后面我们慢慢完善，就发现他的作用了，这里暂时不管。
function resolvePromise(promise2, x, resolve, reject) {
    if (x instanceof MyPromise) {
        // 如果x是promise的实例，则需要分类讨论是否pending
        if (x.status === 'pending') {
            // 如果promise仍然是pending，则注册这个promise的回调函数，继续递归，直到返回值不是pending的promise为止
            x.then(value => {
                resolvePromise(promise2, value, resolve, reject);
            }, reject);
        } else {
            // 如果promise以及被决断，则肯定有个resolve或者reject的值，直接调用then，拿到promise的最终值即可
            x.then(resolve, reject);
        }
    } else {
        // 如果x不是promise，直接resolve，降级为上面的x值非promise的情况
        resolve(x);
    }
}
```
这个函数是整个`MyPromise`中最抽象的地方。我们先分类讨论它的情况。
- 如果`x`的值不是`MyPromise`的实例，问题降级为上面`x`非`promise`的情况。
- 如果`x`的值是`MyPromise`的实例，但是状态已经被`fulfilled`或者`rejected`，则`x`肯定有一个最终的`value`，我们只需继续`x.then(reolve, reject)`，拿到`x`这个`MyPromise`的最终值即可。
- 如果`x`的值是`MyPromise`的实例，但是状态仍然是`pending`，我们同样需要调用`then`函数（`promise`每一步只能去`then`），去注册这个`x`的回调函数（再次啰嗦，`x`是`MyPromise`实例），继续递归，直到返回值不是`pending`状态的`MyPromise`为止。

至此，我们的`MyPromise`基本可用了，贴一下`then`方法此时的样子
```js
MyPromise.prototype.then = function (onFulfilled, onRejected) {
    let self = this;
    let promise2;

    if (self.status === 'fulfilled') {
        return promise2 = new MyPromise(function (resolve, reject) {
            try {
               let x = onFulfilled(self.data);
               // 替换
               resolvePromise(promise2, x, resolve, reject); 
            } catch (e) {
                reject(e);
            }
        });
    } else if (self.status === 'rejected') {
        return promise2 = new MyPromise(function (resolve, reject) {
            try {
                let x = onRejected(self.data);
                // 替换
                resolvePromise(promise2, x, resolve, reject); 
            } catch (e) {
                reject(e)
            }
        });
    } else if (self.status === 'pending') {
        return promise2 = new MyPromise(function (resolve, reject) {
            // 如果是`pending`状态，则一样只需给当前的数组压入回调即可
            self.onFulfilledCallbacks.push((value) => {
                try {
                    let x = onFulfilled(value);
                    // 替换
                    resolvePromise(promise2, x, resolve, reject); 
                } catch (e) {
                    reject(e);
                }
            });
            self.onRejectedCallbacks.push((reason) => {
                try {
                    let x = onRejected(reason);
                    // 替换
                    resolvePromise(promise2, x, resolve, reject); 
                } catch (e) {
                    reject(e);
                }
            });
        });
    }
}
```

组装完成后，配合下面的demo，可以查看

```js
let p = new MyPromise(function (resolve, reject) {
    resolve(100);
});
p.then(function (data) {
    return new MyPromise((resolve, reject) => {
        setTimeout(() => {
            resolve(data + 100);
        }, 2000);
    });
}).then(function (data) {
    console.log(data);
});
```

基本的`Promise`以及实现了，但是仍然有很多细节，我们慢慢补充。
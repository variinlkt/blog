


### Tapable

Compiler 和 Compilation 都继承自 Tapable，可以直接在 Compiler 和 Compilation 对象上广播和监听事件

> webpack 的插件架构主要基于 Tapable 实现的，Tapable 是 webpack 项目组的一个内部库，主要是抽象了一套插件机制。webpack 源代码中的一些 Tapable 实例都继承或混合了 Tapable 类。Tapable 能够让我们为 javaScript 模块添加并应用插件。 它可以被其它模块继承或混合。它**类似于 NodeJS 的 EventEmitter 类，专注于自定义事件的触发和操作**。 除此之外, Tapable 允许你通过回调函数的参数访问事件的生产者。

1.0版本前的Tapable和1.0后的差别较大，区别：

1.0前：Tapable 实例对象只有四组成员函数（plugin，apply，applyPlugins*，mixin），插件可以使用 plugin 方法注入自定义的构建步骤，通过调用插件的 apply 方法来安装它们，并且传递一个 webpack compiler 对象的引用。然后你可以调用 `compiler.plugin` 来访问资源的编译和它们独立的构建步骤：


```
// 1、some-webpack-plugin.js 文件（独立模块）

// 2、模块对外暴露的 js 函数

function SomewebpackPlugin(pluginOpions) {

    this.options = pluginOptions;

}

// 3、原型定义一个 apply 函数，并注入了 compiler 对象

SomewebpackPlugin.prototype.apply = function (compiler) {

    // 4、挂载 webpack 事件钩子（这里挂载的是 emit 事件）

    compiler.plugin('emit', function (compilation, callback) {

        // ... 内部进行自定义的编译操作

        // 5、操作 compilation 对象的内部数据

        console.log(compilation);

        // 6、执行 callback 回调

        callback();

    });

};

// 暴露 js 函数

module.exports = SomewebpackPlugin;
```


1.0后：多了很多成员函数，并且是通过applyPlugins*来触发事件调用


```
const {

    SyncHook,

    SyncBailHook,

    SyncWaterfallHook,

    SyncLoopHook,

    AsyncParallelHook,

    AsyncParallelBailHook,

    AsyncSeriesHook,

    AsyncSeriesBailHook,

    AsyncSeriesWaterfallHook

 } = require("tapable");


//usage

class Order {

    constructor() {

        this.hooks = { //hooks

            goods: new SyncHook(['goodsId', 'number']),

            consumer: new AsyncParallelHook(['userId', 'orderId'])

        }

    }

    queryGoods(goodsId, number) {

        this.hooks.goods.call(goodsId, number);

    }

    consumerInfoPromise(userId, orderId) {

        this.hooks.consumer.promise(userId, orderId).then(() => {

            //TODO

        })

    }

    consumerInfoAsync(userId, orderId) {

        this.hooks.consumer.callAsync(userId, orderId, (err, data) => {

            //TODO

        })

    }

}

let order = new Order()

// 调用tap方法注册一个consument

order.hooks.goods.tap('QueryPlugin', (goodsId, number) => {

    return fetchGoods(goodsId, number);

})

// 再添加一个

order.hooks.goods.tap('LoggerPlugin', (goodsId, number) => {

    logger(goodsId, number);

})

// 调用

order.queryGoods('10000000', 1)
```

![](https://dfiles.tita.com/Portal/110006/fc31163843fa44ce8a80443d83f4c641.png)

![](https://dfiles.tita.com/Portal/110006/0c933bb1d72d4e03a2ba7230b4cc29c8.png)

![](https://dfiles.tita.com/Portal/110006/feba24cce8974b47955be658b25774d9.png)
#### Sync*类型的钩子

注册在该钩子下面的插件的执行顺序都是顺序执行。
只能使用tap注册，不能使用tapPromise和tapAsync注册
#### Async*类型钩子

支持tap、tapPromise、tapAsync注册
每次都是调用tap、tapSync、tapPromise注册不同类型的插件钩子，通过调用call、callAsync 、promise方式调用。

![](https://dfiles.tita.com/Portal/110006/65546fda29f945adb8a5b1aafedb9ef1.png)

不管是`Sync*`类型的钩子还是`Async*`类型钩子，他们都继承于Hook函数

### HOOK函数分析：[源码](https://github.com/webpack/tapable/blob/master/lib/Hook.js)

![](https://dfiles.tita.com/Portal/110006/c9471ebc9c914bc1803ebea55fdff261.png)

里面注册了不同类型的钩子：

![](https://dfiles.tita.com/Portal/110006/a1f5d9b73bc0435cb6834f5f5f3c1d12.png)

注册了tap，async，promise的钩子

![](https://dfiles.tita.com/Portal/110006/4828f4877fc04878815f66b25a752462.png)

这里刚开始看其实不是很懂，给_call,_promise,_callAsync触发事件创建编译委托，调用_createcall函数，调用compile函数，但是compile函数返回一个error？？



emmm，看一下sync类型的钩子


![](https://dfiles.tita.com/Portal/110006/ffc2c4aa739e4ee5963fa695419911b4.png)


sync钩子只能通过tap来订阅，这里也写了compile函数，也就是说上面hook函数的compile会走到这里，当我们触发了事件后会创建编译委托，调用_createcall函数，调用compile函数，调用synchookcodefactory的setup和create函数


![](https://dfiles.tita.com/Portal/110006/e04048738eec41c8a4f709834a34e1ba.png)

setup函数调用taps队列里的函数


![](https://dfiles.tita.com/Portal/110006/8094074f426244dfa53c824f89d040d8.png)

create返回一个封装好的函数

联系usage来看一下：


```
const { SyncHook } = require('tapable');

const mySyncHook = new SyncHook(['name', 'age']);

// 为什么叫tap水龙头 接收两个参数，第一个参数是名称， 第二个参数是一个函数，接受的参数name和上面的name对应，age和上面的age对应

mySyncHook.tap('1', function (name, age) {

    console.log(name, age, 1)

    return 'wrong' // 不关心返回值 这里写返回值对结果没有任何影响

});

mySyncHook.tap('2', function (name, age) {

    console.log(name, age, 2)

});

mySyncHook.tap('3', function (name, age) {

    console.log(name, age, 3)

});

mySyncHook.call('aaa', '18');

// 执行的结果

// aaa 18 1

// aaa 18 2

// aaa 18 3
```

只能调用tap，将传入的两个参数封装成option对象（name，type：sync，fn：函数），调用了hook的`_runRegisterInterceptors`和`_insert`，来注册拦截者，并将调用方法call、callAsync、promise指向内部函数，将注册好的拦截者添加到taps队列中

user调用call函数时，调用_call内部函数，进入高阶函数`createCompileDelegate`的`lazyCompileHook`，`this.call=_createCall()`,调用SyncHook类的compile函数，遍历队列里的函数，获得一个封装好的函数，最后执行这个函数


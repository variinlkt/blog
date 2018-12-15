## webpack：compiler&compilation

本来是想学习一下htmlwebpackplugin 这个插件看下原理，在看的过程中了解到了tapable，后来又了解到了compiler...所以还是先从基础学起吧



webpack最核心的两个对象是compiler 和 compilation 对象，要写webpack插件也必须要了解这两个对象

### compiler 对象：[源码](https://github.com/webpack/webpack/blob/master/lib/Compiler.js)

compiler 对象是 webpack 的编译器对象，compiler 对象会在启动 webpack 的时候被一次性的初始化，compiler 对象中包含了所有 webpack 可自定义操作的配置，例如 loader 的配置，plugin 的配置，entry 的配置等各种原始 webpack 配置等

#### compiler 事件钩子

![](https://dfiles.tita.com/Portal/110006/ed1cdc287e57482caae0f36371e27b1f.png)

![](https://dfiles.tita.com/Portal/110006/fec88b668ab74c21b9278f912fa051ee.png)

![](https://dfiles.tita.com/Portal/110006/fb3e343f188e44388fb61733bca475c8.png)





### compilation 对象 ：[源码](https://github.com/webpack/webpack/blob/master/lib/Compilation.js)

**compilation 对象生成编译资源**。编译资源是 webpack 通过配置生成的一份静态资源管理 Map（一切都在内存中保存），以 key-value 的形式描述一个 webpack 打包后的文件，**编译资源就是这一个个 key-value 组成的 Map。**

**compilation 实例继承于 compiler，compilation 对象代表了一次单一的版本 webpack 构建和生成编译资源的过程。**
> 当运行 webpack 开发环境中间件时，每当检测到一个文件变化，一次新的编译将被创建，从而生成一组新的编译资源以及新的 compilation 对象。一个 compilation 对象包含了 当前的模块资源、编译生成资源、变化的文件、以及 被跟踪依赖的状态信息。编译对象也提供了很多关键点回调供插件做自定义处理时选择使用。
> 
> Compilation 模块会被 Compiler 用来创建新的编译（或新的构建）。**compilation 实例能够访问所有的模块和它们的依赖**（大部分是循环依赖）。它会对应用程序的依赖图中所有模块进行字面上的编译(literal compilation)**。在编译阶段，模块会被加载(loaded)、封存(sealed)、优化(optimized)、分块(chunked)、哈希(hashed)和重新创建(restored)。**
> 

#### compilation钩子

![](https://dfiles.tita.com/Portal/110006/ef873f531fb746c59fcd3db91f7c4543.png)




### 总结：

Compiler 对象包含了 Webpack 环境所有的的配置信息，包含 options，loaders，plugins 这些信息，这个对象在 Webpack 启动时候被实例化，它是全局唯一的，可以简单地把它理解为 Webpack 实例；

Compilation 对象包含了当前的模块资源、编译生成资源、变化的文件等。当 Webpack 以开发模式运行时，每当检测到一个文件变化，一次新的 Compilation 将被创建。Compilation 对象也提供了很多事件回调供插件做扩展。通过 Compilation 也能读取到 Compiler 对象。

#### Compiler 和 Compilation 的区别：

Compiler 代表了整个 Webpack 从启动到关闭的生命周期，而 Compilation 只是代表了一次新的编译。

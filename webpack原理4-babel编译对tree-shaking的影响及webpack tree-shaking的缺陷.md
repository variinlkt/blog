### babel编译对tree-shaking的影响及webpack tree-shaking的缺陷

比如说

![](https://dfiles.tita.com/Portal/110006/045139b48d0243ab952c8a71a92e0c32.png)

当我们将上面的代码用babel编译后：

![](https://dfiles.tita.com/Portal/110006/502a61026f9443b9b9ca5d9cfa72892f.png)



**发现我们的Person类被封装成了一个IIFE，并且tree-shaking失败**



为什么会这样呢？

**因为Webpack Tree shaking不会清除IIFE**

如果我们把square写成iife，即使我们没有引用，webpack也会将它打包出来:

![](https://dfiles.tita.com/Portal/110006/042cebe7aced482db8f5b5103438d5a0.png)

**因为IIFE比较特殊，它在被翻译时(JS并非编译型的语言)就会被执行，Webpack不做程序流分析，它不知道IIFE会做什么特别的事情，所以不会删除这部分代码（只对于没有返回值的iife）**


**对于有返回值的iife，如果未被使用：**

**①iife内部的代码不会清除**

**②return的方法会被清除**


![](https://dfiles.tita.com/Portal/110006/c33d11a938a74252859d67f321a6b97c.png)


### webpack tree-shaking的缺陷1

关于_createClass函数：

![](https://dfiles.tita.com/Portal/110006/158c8cf8297e43af88042805f91b8a5d.png)



可以看到在webpack编译的class是通过Object.defineProperty的形式来对类添加方法



#### 那么Babel为什么要这样去声明构造函数的？

因为ES6的一些语法是有其特定的语义的。比如：

**类内部声明的方法，是不可枚举的，而通过原型链声明的方法是可以枚举的**。

for...of的循环是通过遍历器(Iterator)迭代的，循环数组时并非是i++，然后通过下标寻值。

所以**babel为了符合ES6真正的语义，编译类时采取了Object.defineProperty来定义原型方法，于是导致了后续这些一系列问题。**


#### 如何使webpack将class 编译成直接在原型链上声明方法？

babel其实是有一个**loose模式**的，直译的话叫做宽松模式。它是做什么用的呢？**它会不严格遵循ES6的语义，而采取更符合我们平常编写代码时的习惯去编译代码。**


```
// .babelrc

{"presets": [

    ["env", {
    
        “loose”: true
    
    }]

]}
```


对于tree-shaking来说，这样也**仅仅只能去除_createClass，Person本身依旧存在**



#### 为什么使用了loose还是tree-shaking失败呢？

因为**uglify不进行程序流分析，所以不能排除有可能有副作用的代码**

> 函数的参数若是引用类型，对于它属性的操作，都是有可能会产生副作用的。因为首先它是引用类型，对它属性的任何修改其实都是改变了函数外部的数据。其次获取或修改它的属性，会触发getter或者setter，而getter、setter是不透明的，有可能会产生副作用。
> 
> uglify没有完善的程序流分析。它可以简单的判断变量后续是否被引用、修改，但是不能判断一个变量完整的修改过程，不知道它是否已经指向了外部变量，所以很多有可能会产生副作用的代码，都只能保守的不删除。
> 
> rollup有程序流分析的功能，可以更好的判断代码是否真正会产生副作用。


#### 如果想使用babel又想tree-shaking又要用uglify又有class代码的情况该肿么办？

网上查了，说让使用babel7，然后在iife内部或头部添加`/*@__PURE__*/`或`/*#__PURE__*/`注释

亲测没用

尝试了**下BabelMinifyWebpackPlugin代替uglify，有效**



#### 总结：如果要更好的使用Webpack Tree shaking,请满足：

- 使用ES2015(ES6)的模块
- 避免使用IIFE
- 如果使用第三方的模块，可以尝试直接从文件路径引用的方式使用（这并不是最佳的方式）

```
import { fn } from 'module'; 

=> 

import fn from 'module/XX';
```

- **避免使用babel+uglify**，使用babel-minify或BabelMinifyWebpackPlugin代替uglify
- 用rollup吧


### webpack tree-shaking的缺陷2

![](https://dfiles.tita.com/Portal/110006/3f815011ef6947b39dfa71e1f5669264.png)

这个array函数未被使用，但是lodash-es这个包的部分代码还是会被build到bundle.js中

解决方法：

**可以使用这个插件webpack-deep-scope-analysis-plugin解决**

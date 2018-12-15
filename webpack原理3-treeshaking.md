### webpack tree-shaking
#### 原理
> - ES6的模块引入是静态分析的，故而可以在编译时正确判断到底加载了什么代码。
> - 分析程序流，判断哪些变量未被使用、引用，进而删除此代码。
> 所以一旦 babel 将 es6 的模块转换成 commonjs，webpack2 将无法使用这项优化。所以要使用这项技术，我们只能使用 webpack 的模块处理，加上 babel 的es6转换能力（需要关闭模块转换）。

当我开开心心将babel-loader按如下的配置配置了以后，发现并不行：

```
{

    test:/\.js$/,

    exclude: /node_modules/, 

    use:[

      {

        loader:'babel-loader',

        options:{

          "presets": [

            ["env", {

              "modules": false

            }]

          ]

        }

      }

    ]

  },
```

最后折腾了大半天，终于发现：要把option那些配置加到.babelrc文件里面才有效：

```
//.babelrc

"presets": [

    ["env", {

      "modules": false

    }]

  ]
```

tree-shaking的小小总结：

1、mode为production

2、babel的option配置要写入.babelrc文件

#### 为什么要用production mode

可以从[官网文档](https://webpack.js.org/concepts/mode/#usage)中看到两种mode的配置情况


以下是production的时候的配置情况：

![](https://dfiles.tita.com/Portal/110006/609a7f1e557846819e55311f5c00a114.png)


以下是development的配置情况：

![](https://dfiles.tita.com/Portal/110006/37198342433b4a049ed7b22ed6a70d96.png)




可以看到以下几个关键的区别：

![](https://dfiles.tita.com/Portal/110006/a97085c968d74e33ad553da10b1f1251.png)




然后来对比几个比较关键的区别：

**1、devtool**


```
//devtool:false：

/*!******************!*\

  !*** ./src/b.js ***!

  \******************/

/*! exports provided: square, cube */

/*! exports used: cube */function(e,t,r){"use strict";function n(e){return e*e*e}r.d(t,"a",function(){return n})},"./src/index.js":



//devtool:’eval’：

/*!******************!*\

  !*** ./src/b.js ***!

  \******************/

/*! exports provided: square, cube */

/*! exports used: cube */function(module,__webpack_exports__,__webpack_require__){"use strict";eval('/* unused harmony export square */\n/* harmony export (binding) */ __webpack_require__.d(__webpack_exports__, "a", function() { return cube; });\nfunction square(x) {\n  return x * x;\n}\n\nfunction cube(x) {\n  return x * x * x;\n}\n\n//# sourceURL=webpack:///./src/b.js?')},"./src/index.js":
```


可以看到，在minimize：true+devtool:eval+usedExport：true的情况下**虽然能标识出square是unused harmony export，但是最后还是没能将square函数的声明清除**



**2、usedExport**


```
//usedExports:false：

/*!******************!*\

  !*** ./src/b.js ***!

  \******************/

/*! exports provided: square, cube */function(e,r,t){"use strict";function n(e){return e*e}function u(e){return e*e*e}t.r(r),t.d(r,"square",function(){return n}),t.d(r,"cube",function(){return u})},"./src/index.js":
```



可以看到，在minimize：true+devtool：false+usedExport：false的情况下**还是将square export出来，甚至根本没识别出它是unused的**



那usedExports是什么？

> Dead code elimination in minimizers will benefit from this and can remove unused exports. By default optimization.usedExports is enabled in production mode and disabled elsewise.

意思是说，当usedExports为true的时候才会将死代码（即unused export）消除，可以说是tree-shaking的核心配置了



**3、minimize**

![](https://dfiles.tita.com/Portal/110006/ee6adf37bf0f4ca7b4a109f83afcfb43.png)



可以看到，**不压缩代码时不会将没使用的export的函数代码删除**

#### 总结：

##### 0、tree-shaking的过程：

- 如果使用babel，在babel配置，防止让es6语法转换为require/module.exports：


```
// .babelrc

{"presets": [
["env", { "modules": false}]
]}
```


- 如果不使用babel，那么配置：


```
optimization:{

    concatenateModules: true,

}
```


```
plugins:[

    new webpack.optimize.ModuleConcatenationPlugin()

]
```


来尝试找到模块间的关联关系并将可以合并的模块合并掉

- usedExports先将那些需要删除的“未使用代码(dead code)”找出来

- 通过设置minimize启用压缩，在 bundle 中删除dead code



##### 1、tree-shaking的必要条件

- usedExport:true

- devtool:false

- minimize:true

##### 2、为什么说要将mode设置为production？

因为production时就已经默认将tree-shaking的必要条件配置好了



demo:https://github.com/variinlkt/webpack-tree_shaking-demo

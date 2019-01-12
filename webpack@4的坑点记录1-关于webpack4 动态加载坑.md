# webpack@4的坑点记录1-关于webpack4 动态加载dynamic imports的坑

当我们在项目中应用到一些比较庞大的组件的时候，可以将那些庞大的组件进行单独打包

单独打包其实有好几种方法：

1、可以在entry手动指定分割，缺点是需要手动配置分割，并且重复的模块可能会打包进多个 bundle 中，无法去重

不过可以使用optimization.splitChunks提取重复模块：

```
optimization: {

    // runtimeChunk: {

    //     name: "manifest"

    // },

    splitChunks: {

        cacheGroups: {

            commons: {

                chunks: 'initial',

                minChunks: 2,

                maxInitialRequests: 5,

                minSize: 0

            },

            vendor: { // 将第三方模块提取出来

                test: /node_modules/,

                chunks: 'initial',

                name: 'vendor',

                priority: 10, // 优先

                enforce: true

            }

        }

    }

}
```


但是之前也做过这种分离代码的方法，想玩点新的。。

最近尝试了下webpack dynamic imports

动态加载有两种方法，一种是require.ensure，一种是import()

1、require.ensure使用：

```
if (component.name == 'OceanTheme') {

              

                require.ensure([],function(require){

                    require('Components/OceanTheme/controller')

                    console.log('ok')

                },function(e){

                    console.error(e)

                },'ocean')

            }

```
我们等下再来看下打包结果

2、import()使用：

```
const r = import(/* webpackChunkName: "ocean" */ 'Components/OceanTheme/controller')

//webpack.config.js

output:{

chunkFilename: '[name].bundle.js',

}

```

理想很丰满，然而现实是。。。

编译报错了：
![](https://dfiles.tita.com/Portal/110006/457683da4382458aadb1f7a01c0ba18a.png)


google查了下，安装了个包：`@babel/plugin-syntax-dynamic-import`

文档：https://babeljs.io/docs/en/babel-plugin-syntax-dynamic-import

```
//.babelrc

{

  "plugins": ["@babel/plugin-syntax-dynamic-import"]

}

```

但是还是没有分离出独立的bundle

又查资料：https://juejin.im/post/5ba654c4e51d450e3d2cdd83

才发现在webpack4下import()是有“bug”的（在webpack3下正常）：
> 打包过程发生在编译阶段，webpack看不懂类似import(someFunction())这种语法，因为在编译阶段不知道someFunction()是什么， webpack只认识这种语法import('./app' + xx + '/store/index.js')，webpack的理解就是 把变量替换成占位符，把匹配这个模式./app/*/store/index.js的文件分别打成chunk，然后程序运行的时候，再根据变量生成的路径，匹配对应chunk块。

>有效的变量拼接开头
>
>1、以webpack配置的alias开头 


```
alias: {     Root: 'e:/xx', } 

import('Root/' + args + 'index.js)
```


> 2、以./开头的语法 `import('Root/' + args + 'index.js)`
> 
> 3、诸如../和绝对路径开头都无效。



**经过实践，发现在webpack@4.27.1中，require.ensure和import里面只能支持“./”开头的路径，alias无效(绝对路径)**

**类似“./../components/…”这种也无效**

## 结论
- 在webpack@4.27.1中，require.ensure和import里面只能支持“./”开头的路径，alias无效(绝对路径)
- 以上问题在webpack@3中不存在
- 如果要用动态加载，要么降级webpack，要么将加载的组件写到当前层级的路径下

### babel在webpack编译的作用

babel 能提前将 es6 的 import 等模块关键字转换成 commonjs 的规范。这样 webpack 就无需再做处理，直接使用 webpack 运行时定义的 __webpack_require__ 处理。



#### babel 是如何转换 es6 的模块语法呢？

##### 1、对于导出模块

es6 的导出模块写法有：


```
export default 123;

export const a = 123;

const b = 3;

const c = 4;

export { b, c };
```


babel 会将这些统统转换成 commonjs 的 exports。


```
exports.default = 123;

exports.a = 123;

exports.b = 3;

exports.c = 4;

exports.__esModule = true;
```


babel将模块的导出转换为commonjs规范后，也会将引入 import 也转换为 commonjs 规范。



##### 2、对于引入 default模块

对于最常见的


```
import a from './a.js';
```


在es6中 import a from './a.js' 的本意是想去引入一个 es6 模块中的 default 输出。

通过babel转换后得到 var a = require(./a.js) 得到的对象却是整个对象，肯定不是 es6 语句的本意，所以需要对 a 做些改变。

default 输出会赋值给导出对象的default属性。

exports.default = 123;

所以 babel 加了个 help _interopRequireDefault 函数。


```
function _interopRequireDefault(obj) {

    return obj && obj.__esModule

        ? obj

        : { 'default': obj };

}


var _a = require('assert');

var _a2 = _interopRequireDefault(_a);

var a = _a2['default'];
```

所以这里最后的 a 变量就是 require 的值的 default 属性。如果原先就是commonjs规范的模块，那么就是那个模块的导出对象。

因此es6对于引入default模块是通过调用了_interopRequireDefault函数获取了require到的对象的default属性，从而避免引入整个模块对象



##### 3、对于引入*通配符


```
import * as a from './a.js'
```


es6语法的本意是想将 es6 模块的所有命名输出以及defalut输出打包成一个对象赋值给a变量。

已知以 commonjs 规范导出：


```
exports.default = 123;

exports.a = 123;

exports.b = 3;

exports.__esModule = true;
```


那么对于 es6 转换来的输出通过 var a = require('./a.js') 导入这个对象就已经符合意图。

所以直接返回这个对象。


```
if (obj && obj.__esModule) {

   return obj;

}
```

如果本来就是 commonjs 规范的模块，导出时没有default属性，需要添加一个default属性，并把整个模块对象再次赋值给default属性。



##### 4、对于引入{}模块


```
import { a } from './a.js'
```


直接转换成 require('./a.js').a 即可。

即使我们使用了 es6 的模块系统，如果借助 babel 的转换，es6 的模块系统最终还是会转换成 commonjs 的规范。所以我们如果是使用 babel 转换 es6 模块，混合使用 es6 的模块和 commonjs 的规范是没有问题的，因为最终都会转换成 commonjs



#### 为何有的地方使用 require 去引用一个模块时需要加上 default？ require('xx').default

因为：


```
// a.js

export default 123;

//对于以上代码会转换为：

exports.default = 123;

// b.js 错误

var foo = require('./a.js')

// 正确

var foo = require('./a.js').default
```


ps：该变更属于babel6的变更：

在 babel5 时代，大部分人在用 require 去引用 es6 输出的 default，只是把 default 输出看作是一个模块的默认输出，所以 babel5 对这个逻辑做了 hack，如果一个 es6 模块只有一个 default 输出，那么在转换成 commonjs 的时候也一起赋值给 module.exports，即整个导出对象被赋值了 default 所对应的值。





#### webpack 编译后的js，如何再被其他模块引用

通过 webpack 模块化原理章节给出的 webpack 配置编译后的 js 是无法被其他模块引用的

webpack 提供了 output.libraryTarget 配置指定构建完的 js 的用途。

1、libraryTarget: "var" 默认

认当 library 加载完成，入口起点的返回值将分配给一个变量：

如果指定了 output.library = 'test' 入口模块返回的 module.exports 暴露给全局


```
var test = returned_module_exports
```


2、libraryTarget: "commonjs"

如果library: 'spon-ui' 入口模块返回的 module.exports 赋值给 exports['spon-ui']

3、libraryTarget: "commonjs2"

入口模块返回的 module.exports 赋值给 module.exports

所以 element-ui 的构建方式采用 commonjs2 ，导出的组件的js 最后都会赋值给 module.exports，供其他模块引用。



### 按需加载的原理

我们在使用各大 UI 组件库时都会被介绍到为了避免引入全部文件，请使用 babel-plugin-component 等babel 插件。


```
import { Button, Select } from 'element-ui'
```



由前文可知 import 会先转换为 commonjs， 即


```
var a = require('element-ui');

var Button = a.Button;

var Select = a.Select;



var a = require('element-ui');
```
 这个过程就会将所有组件都引入进来了。

所以 babel-plugin-component就将 import { Button, Select } from 'element-ui' 转换成了


```
import Button from 'element-ui/lib/button'

import Select from 'element-ui/lib/select'
```


即使转换成了 commonjs 规范，也只是引入自己这个组件的js，将引入量减少到最低。

所以我们会看到几乎所有的UI组件库的目录形式都是

```
|-lib

||--component1

||--component2

||--component3

|-index.common.js
```



index.common.js 给 import element from 'element-ui' 这种形式调用全部组件。

lib 下的各组件用于按需引用。



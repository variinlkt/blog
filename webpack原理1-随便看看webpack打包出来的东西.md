### webpack原理1

首先我先看了下webpack打包出来的东西


```
//入口文件：

import greet from './a'

greet()

//a.js

import greet from './b'

export default function greet2someone(){

    console.log(greet()+'someone')

}

//b.js

export default function greet(){

    return 'hi'

}
```


webpack编译后：

生成一个iife


```
(function(modules) {

})({});
```


Iife内部有一个__webpack_require__函数：


```
/******/function __webpack_require__(moduleId) {

/******/    // Check if module is in cache

/******/    if(installedModules[moduleId]) {

/******/      return installedModules[moduleId].exports;

/******/    }

/******/    // Create a new module (and put it into the cache)

/******/    var module = installedModules[moduleId] = {

/******/      i: moduleId,

/******/      l: false,

/******/      exports: {}

/******/    };

/******/    // Execute the module function

/******/    modules[moduleId].call(module.exports, module, module.exports, __webpack_require__);

/******/    // Flag the module as loaded

/******/    module.l = true;

/******/    // Return the exports of the module

/******/    return module.exports;

/******/  }
```


这里执行了：


```
modules[moduleId].call(module.exports, module, module.exports, __webpack_require__);
```


即从入参的 modules 数组中取"./src/index.js”的key，即入口文件进行调用，并入参

再看看它对应的函数：


```
/***/ (function(module, exports, __webpack_require__) {

"use strict";

eval("\n\nvar _a = __webpack_require__(/*! ./a */ \"./src/a.js\");\n\nvar _a2 = _interopRequireDefault(_a);\n\nfunction _interopRequireDefault(obj) { return obj && obj.__esModule ? obj : { default: obj }; }\n\n(0, _a2.default)();\n\n//# sourceURL=webpack:///./src/index.js?");

/***/ })
```


相当于：


```
(function(module, exports, __webpack_require__) {

"use strict";

var _a = __webpack_require__("./src/a.js");

var _a2 = _interopRequireDefault(_a);

function _interopRequireDefault(obj) { 

  return obj && obj.__esModule ? obj : { default: obj }; 

}

(0, _a2.default)();//# sourceURL=webpack:///./src/index.js

})
```


可以看到入口模块调用了__webpack_require__去引入了a.js

然后将模块传入了_interopRequireDefault函数，这个函数是看这个模块是否有__esModule属性，__esModule 表明这是个由 es6 转换来的 commonjs 输出



#### 那么这个__esModule属性的作用是什么呢？

给模块的输出对象增加 __esModule 是为了**将不符合 Babel 要求的 CommonJS 模块转换成符合要求的模块**，这一点在 require 的时候体现出来。

如果加载模块之后，发现加载的模块带着一个 __esModule 属性，Babel 就知道这个模块肯定是它转换过的，这样 Babel 就可以放心地从加载的模块中调用 exports.default 这个导出的对象，也就是 ES6 规定的默认导出对象，所以这个模块既符合 CommonJS 标准，又符合 Babel 对 ES6 模块化的需求。

然而如果 __esModule 不存在，也没关系，Babel 在加载了一个检测不到 __esModule 的模块时，它就知道这个模块虽然符合 CommonJS 标准，但可能是一个第三方的模块，Babel 没有转换过它，如果以后直接调用 exports.default 是会出错的，**所以现在就给它补上一个 default 属性**，就干脆让 default 属性指向它自己就好了，这样以后就不会出错了。



#### (0, _a2.default)()这句代码是什么意思呢？

这句话的意思是：如果执行 `(0, foo.bar)()`，这个逗号表达式等价于执行 `foo.bar()`，但是执行时的上下文环境会被绑定到全局对象身上，所以实际上真正等价于执行 `foo.bar.call(GLOBAL_OBJECT)`。



#### 为什么this会绑定到全局呢？

`(0, _a2.default)`是一个小括号表达式,小括号表达式会依次创建两个匿名函数,并返回最后一个的匿名函数值

也就是相当于：


```
var foo=_a2.default

foo();
```


那么this就指向全局了，针滴巧妙



传入的参数对象：


```
{

/***/ "./src/a.js":

/***/ (function(module, exports, __webpack_require__) {

"use strict";

eval("\n\nObject.defineProperty(exports, \"__esModule\", {\n    value: true\n});\nexports.default = greet2someone;\n\nvar _b = __webpack_require__(/*! ./b */ \"./src/b.js\");\n\nvar _b2 = _interopRequireDefault(_b);\n\nfunction _interopRequireDefault(obj) { return obj && obj.__esModule ? obj : { default: obj }; }\n\nfunction greet2someone() {\n    console.log((0, _b2.default)() + 'someone');\n}\n\n//# sourceURL=webpack:///./src/a.js?");

/***/ }),

/***/ "./src/b.js":

/***/ (function(module, exports, __webpack_require__) {

"use strict";

eval("\n\nObject.defineProperty(exports, \"__esModule\", {\n    value: true\n});\nexports.default = greet;\nfunction greet() {\n    return 'hi';\n}\n\n//# sourceURL=webpack:///./src/b.js?");

/***/ }),

/***/ "./src/index.js":

/***/ (function(module, exports, __webpack_require__) {

"use strict";

eval("【代码上面写过了】");

/***/ })

/******/ }
```


展开eval函数：


```
//a.js

Object.defineProperty(exports, "__esModule", {

  value: true

});

exports.default = greet2someone;

var _b = __webpack_require__("./src/b.js");

var _b2 = _interopRequireDefault(_b);

function _interopRequireDefault(obj) { 

  return obj && obj.__esModule ? obj : { default: obj }; 

}

function greet2someone(someone) {    

  console.log((0, _b2.default)() + someone);

}

//b.js

Object.defineProperty(exports, "__esModule", {

value: true

});

exports.default = greet;

function greet() {    

return 'hi';

}
```

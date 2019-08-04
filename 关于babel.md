# babel7
最近遇到了升级babel7的坑，费了好多时间定位并解决了问题，这里也记录一下遇到的问题以及对babel（7）的总结。
## 问题场景
公司一些老项目引入了一些自己打包成模块的组件，但是引入的组件并没有转译成es5，导致在一些低端机型会出现白页、报错const的问题。

乍一看好像挺好解决的，在配置文件中include进引入组件的对应的node_modules包就好了，然而实际情况是不管怎么在babelrc文件中配置，打包出来的文件里面照样一堆es6语法。

听上去还挺让人摸不着头脑的，在文中会给出解答，现在先来看下在使用babel中需要了解的东西吧（噢看我这糟糕的文笔）

## config配置
在babel7中，除了可以用.babelrc文件配置，还可以用babel.config.js文件进行配置.

两种配置是平行的，可以同时使用

### .babelrc ——文件相对配置
通过搜索，Babel 加载 .babelrc文件，目录结构从正在编译的 "filename" 开始

特点：
- 一旦找到包含 package.json 的目录，搜索将停止，因此是相对配置 仅适用于单个包。
- 正在编译的 "filename" 必须有 "babelrcRoots" 包，否则将完全跳过搜索。因为在默认情况下，babel只会搜索在根目录下的.babelrc文件，babelrcRoots这个配置项允许用户提供其他可以作为根目录的包

### babel.config.js ——项目范围的配置
babel7有了“根目录”的概念，默认是当前工作的目录。

在Project-wide配置中，babel会自动在根目录下搜索babel.config.js文件

用户可以通过设置configFile的值来重写配置文件名称

### 复杂场景

```
.babelrc
packages/
  mod1/
    package.json
    src/index.js
  mod2/
    package.json
    src/index.js
```
对于上面这样的目录结构，因为这样的目录结构是跨包的，如果用了babelrc来配置是无效的

#### 解决方法
##### extends
在mod1、mod2文件夹下配置：
```
{ "extends": "../../.babelrc" }
```
这样处理的方式很麻烦
##### babel.config.js
babel.config.js是项目范围的配置，用它来代替babelrc

现在你大概知道文章开头问题场景的原因和解决方法了吧

<s>好了今天我们就总结到这里吧</s>

## 执行顺序
* 插件在 Presets 前运行。
* 插件顺序从前往后排列。
* Preset 顺序是颠倒的（从后往前）。原因：因为大多数用户将 "es2015" 排在 "stage-0" 之前

## presets预设
### @babel/preset-env
取代了以前的es201x，但是并不支持stage-x。
#### es201x
将es(x+1)的语法转换为esx的语法
#### stage-x
es7不同阶段语法提案的转码规则模块（共有4个阶段），分别是stage-0，stage-1，stage-2，stage-3。
* Stage 0   设想（Strawman）：只是一个想法，可能有 Babel插件。
* Stage 1   建议（Proposal）：这是值得跟进的。
* Stage 2   草案（Draft）：初始规范。
* Stage 3   候选（Candidate）：完成规范并在浏览器上初步实现。
* Stage 4   完成（Finished）：将添加到下一个年度版本发布中。

具体就不多介绍了，反正都被废弃了。

#### browserslist
使用了@babel/preset-env，我们可以通过 [.browserslistrc](https://github.com/browserslist/browserslist) 文件来指定特定的目标浏览器，也可以通过targets选项的browsers选项指定。不过如果你的目标浏览器支持 es modules 特性，browsers 选项则会失效

优先级规则是 .babelrc文件定义了则会忽略 browserslist、.babelrc 没有定义则会搜索 browserslist 和 package.json， 两者应该只定义一个，否则会报错。

#### 主要参数


```
"presets": [
    "@babel/preset-react",
    [
      "@babel/preset-env", {
        "modules": false,

        "targets": {
          "browsers": ["Android >= 4.4.0", "ios >= 9.0"]
        },
        "useBuiltIns": "usage",
        "debug": false
      }
    ]
  ],


```

- targets


- targets.node


- targets.browsers


- spec : 启用更符合规范的转换，但速度会更慢，默认为 false


- loose：是否使用 loose mode，默认为 false


- modules：将 ES6 module 转换为其他模块规范，可选 "adm" | "umd" | "systemjs" | "commonjs" | "cjs" | false，默认为 false


- debug：启用debug，默认 false


- include：一个包含使用的 plugins 的数组


- exclude：一个包含不使用的 plugins 的数组


- useBuiltIns：为 polyfills 应用 @babel/preset-env ，可选 "usage" | "entry" | false，默认为 false

##### loose
许多Babel的插件有两种模式：

• 尽可能符合ECMAScript6语义的normal模式。
• 提供更简单ES5代码的loose模式。
通常，推荐不使用loose模式，使用这种模式的优点和缺点是：

• 优点：生成的代码可能更快，对老的引擎有更好的兼容性，代码通常更简洁，更加的“ES5化”，更像是人写的代码。

以ES6的class为例，我们编写以下代码：
```
class person {
  constractor(name, age) {
    this.name = name;
    this.age = age;
  }
  getName(){
    return this.name;
  }
  getAge(){
      return this.age;
  }
}
```
normal mode 下转换为：
```
"use strict";

var _createClass = function () { function defineProperties(target, props) { for (var i = 0; i < props.length; i++) { var descriptor = props[i]; descriptor.enumerable = descriptor.enumerable || false; descriptor.configurable = true; if ("value" in descriptor) descriptor.writable = true; Object.defineProperty(target, descriptor.key, descriptor); } } return function (Constructor, protoProps, staticProps) { if (protoProps) defineProperties(Constructor.prototype, protoProps); if (staticProps) defineProperties(Constructor, staticProps); return Constructor; }; }();

function _classCallCheck(instance, Constructor) { if (!(instance instanceof Constructor)) { throw new TypeError("Cannot call a class as a function"); } }

var person = function () {
  function person() {
    _classCallCheck(this, person);
  }

  _createClass(person, [{
    key: "constractor",
    value: function constractor(name, age) {
      this.name = name;
      this.age = age;
    }
  }, {
    key: "getName",
    value: function getName() {
      return this.name;
    }
  }, {
    key: "getAge",
    value: function getAge() {
      return this.age;
    }
  }]);

  return person;
}();
```
在normal模式下，类的prototype 方法是通过Object.defineProperty 添加的，来确保它们是不可以被枚举的，这是ES6规范所要求的。



loose mode 下转换为：
```
"use strict";

function _classCallCheck(instance, Constructor) { if (!(instance instanceof Constructor)) { throw new TypeError("Cannot call a class as a function"); } }

var person = function () {
  function person() {
    _classCallCheck(this, person);
  }

  person.prototype.constractor = function constractor(name, age) {
    this.name = name;
    this.age = age;
  };

  person.prototype.getName = function getName() {
    return this.name;
  };

  person.prototype.getAge = function getAge() {
    return this.age;
  };

  return person;
}();

```


• 缺点：随后从转译的ES6到原生的ES6时可能会遇到问题。

##### useBuiltIns

###### useBuiltIns: 'entry'
该选项需要在项目中引入 @babel/polyfill, babel 会自动将 @babel/polyfill 分解为更小的、仅目标环境需要的 polyfill 引用
首先要在源码第一行添加 polyfill 引用

```
import '@babel/polyfill'
```
修改 babel 配置

```
// babel 配置
{
    "presets": [
        [
            "@babel/preset-env",
            {
                "targets": "Chrome 40",
                "useBuiltIns": "entry"
            }
        ]
    ]
}
```


编译后的代码自动引入的 Chrome 40 不支持的所有内容, 包括 Set 和 findIndex(), 它并不会去分析源码用到的哪些内容
###### useBuiltIns: 'usage'
它会分析代码调用, 但是**对于原型链上的方法仅仅按照方法名去匹配**, 可以得到更小的 polyfill 体积. 但是它不会去分析代码依赖的 npm 包的内容, 如果某个 npm 包是需要一些 polyfill 的, 那这些 polyfill 并不会被打包进去

这个配置不需要显式引入polyfill，但是要安装

为什么原型链上的方法不能根据是否用到, 然后按需去 polyfill 呢?
主要是**因为 JavaScript 动态类型的特性**, 有些变量/实例的类型是运行时才能确定的, 而 babel 仅仅是对代码的静态编译, 因此它并不能确定 findIndex() 到底是不是 Array.protoptype.findIndex()

TypeScript 具备静态类型, 但是也不能按需 polyfill 

TypeScript 可以用 --lib 参数指定要依赖的库, 搭配 ts-polyfill 可以对依赖的库进行 polyfill, 但是指定依赖时不能详细到某个方法, 只能 ESNext.Array



## plugin插件
### @babel/polyfill
babel-polyfill 是为了模拟一个完整的ES2015+环境，旨在用于应用程序而不是库/工具。并且使用babel-node时，这个polyfill会自动加载。这里要注意的是**babel-polyfill是一次性引入你的项目中的，并且同项目代码一起编译到生产环境**。而且**会污染全局变量**。像Map，Array.prototype.find这些就存在于全局空间中。
#### 使用
安装到生产环境


```
npm install @babel/polyfill --save
```

webpack.config.js中这样配置即可


```
module.exports = {
  entry: ["@babel/polyfill", "./app/js"],
};
```
【此处有大佬指正：这样配置并不好，业务层、编译层一定要分离开】

### @babel/plugin-transform-runtime
存在的意义是：
- babel会插入一些帮助函数比如_extend，默认情况下会插入多个文件中，相当于多个文件里都有个_extend函数声明，这是没必要的；而这个插件会在需要使用帮助函数的文件中引入`@babel/runtime`
- 创建沙盒环境。使用了polyfill会导致全局变量被污染，如果你要开发一个库的话可能会导致问题
只能在开发环境中使用，所以安装在development dependency中

```
npm install --save-dev @babel/plugin-transform-runtime
```

```
{
  "plugins": [
    [
      "@babel/plugin-transform-runtime",
      {
        "absoluteRuntime": false,
        "corejs": false,
        "helpers": true,
        "regenerator": true,
        "useESModules": false
      }
    ]
  ]
}
```
- corejs：默认false，或者数字：{ corejs: 2 }，代表需要使用corejs的版本。
- helpers：默认是true，是否替换helpers。
- regenerator：默认true，generator是否被转译成用regenerator runtime包装不污染全局作用域的代码。
- useESModules：默认false，如果是true将不会用@babel/plugin-transform-modules-commonjs进行转译，这样会减小打包体积，因为不需要保持语义。

#### transform-runtime做了什么
- core-js aliasing：自动导入babel-runtime/core-js，并将全局静态方法、全局内置对象 映射到对应的模块。
- Helper aliasing：将内联的工具函数移除，改成通过babel-runtime/helpers模块进行导入，比如_classCallCheck工具函数。
- Regenerator aliasing：如果你使用了 async/generator 函数，则自动导入 babel-runtime/regenerator模块。


它们都可以通过配置进行开关，默认配置如下：


```
{
  "plugins": [
    ["transform-runtime", {
      "helpers": true,
      "polyfill": true,
      "regenerator": true,
      "moduleName": "babel-runtime"
    }]
  ]
}
```


举个例子：
```
function* foo() {}

//没用runtime transformer时的转换结果
"use strict";

var _marked = [foo].map(regeneratorRuntime.mark);

function foo() {
  return regeneratorRuntime.wrap(
    function foo$(_context) {
      while (1) {
        switch ((_context.prev = _context.next)) {
          case 0:
          case "end":
            return _context.stop();
        }
      }
    },
    _marked[0],
    this
  );
}

// 用了以后
"use strict";

var _regenerator = require("@babel/runtime/regenerator");

var _regenerator2 = _interopRequireDefault(_regenerator);

function _interopRequireDefault(obj) {
  return obj && obj.__esModule ? obj : { default: obj };
}

var _marked = [foo].map(_regenerator2.default.mark);

function foo() {
  return _regenerator2.default.wrap(
    function foo$(_context) {
      while (1) {
        switch ((_context.prev = _context.next)) {
          case 0:
          case "end":
            return _context.stop();
        }
      }
    },
    _marked[0],
    this
  );
}
```
#### runtime怎么做到不污染全局变量的

我觉得吧就是因为设置了{corejs：2}，为代码创建一个沙盒环境，为 core-js 提供假名，这样就做到了不污染全局空间。<s>【待验证】</s>
【已验证】在没有设置corejs时runtime并没有生效，es6的方法并没有转换，只有在设置了corejs后才有效，demo见:https://github.com/variinlkt/babel-demo



#### corejs
corejs 是一个给低版本的浏览器提供接口的库，如 Promise, map, set 等。
在 babel 中你设置成 false 或者不设置，就是引入的是 corejs 中的库，而且在全局中引入，也就是说侵入了全局的变量。


如果你的全局有一个引入，不要让引入的库影响全局，那你就需要引把 corejs 设置成 2。

如果你设置了 corejs2，那你就需要加入下面的库:


```
npm install @babel/runtime-corejs2
```


 Babel 7.4.0 之后，你可以选择引入 @babel/runtime-corejs3，设置 corejs: 3 来帮助您实现对实例方法的支持
 
 corejs: false 其实等同于使用 @babel/polyfill 时的 useBuiltIns: false，只对ES语法进行了转换。corejs：2 等同于 Babel 6时的 polyfill: true ，它们都会为代码创建一个沙盒环境，为 core-js 提供假名，这样就做到了不污染全局空间。corejs: 3 是在 corejs: 2的基础上进而解决了之前无法实例方法的窘况，同时也保持了不污染全局空间


```
// corejs: false
"use strict";

var a = new Promise();
[1, 2, 3].includes(1);

// corejs: 2
"use strict";

var _interopRequireDefault = require("@babel/runtime-corejs2/helpers/interopRequireDefault");

var _promise = _interopRequireDefault(require("@babel/runtime-corejs2/core-js/promise"));

var a = new _promise["default"]();
[1, 2, 3].includes(1);

// corejs: 3
"use strict";

var _interopRequireDefault = require("@babel/runtime-corejs3/helpers/interopRequireDefault");

var _includes = _interopRequireDefault(require("@babel/runtime-corejs3/core-js-stable/instance/includes"));

var _promise = _interopRequireDefault(require("@babel/runtime-corejs3/core-js-stable/promise"));

var _context;

var a = new _promise["default"]();
(0, _includes["default"])(_context = [1, 2, 3]).call(_context, 1);


```
corejs：2 等同于 Babel 6时的 polyfill: true ，它们都会为代码创建一个沙盒环境，为 core-js 提供假名，这样就做到了不污染全局空间。
#### 缺点
**不模拟实例方法，即内置对象原型上的方法**

对于"foobar".includes("foo")这样的实例方法仍然是不能正常执行的，因为他在挂载在String.prototype上的，如果需要使用这样的实例方法，还是不得不require('@babel/polyfill')

### @babel/runtime 
可以在生产环境中使用，存在意义跟@babel/plugin-transform-runtime一样

```
npm install --save @babel/runtime
```

### 什么时候用babel-polyfill，什么时候用babel-runtime
babel-polyfill会污染全局空间，并可能导致不同版本间的冲突，而babel-runtime不会。从这点看应该用babel-runtime。
但记住，7.4.0版本以下的babel-runtime有个缺点，它不模拟实例方法，即内置对象原型上的方法，所以类似Array.prototype.find，你通过babel-runtime是无法使用的。**7.4.0以上的可以通过搭配配置corejs：3来支持内置对象原型上的方法**

### @babel/register
babel-register模块改写require命令，为它加上一个钩子。此后，每当使用require加载.js、.jsx、.es和.es6后缀名的文件，就会先用Babel进行转码。

需要注意的是，babel-register只会对require命令加载的文件转码，而不会对当前文件转码。另外，由于它是实时转码，所以只适合在开发环境使用。
## @babel/core
如果某些代码需要调用Babel的API进行转码，就要使用@babel/core模块

```
 var babel = require('@babel/core');
 
 // 字符串转码
 babel.transform('code();', options);
 // => { code, map, ast }
 
 // 文件转码（异步）
 babel.transformFile('filename.js', options, function(err, result) {
   result; // => { code, map, ast }
 });
 
 // 文件转码（同步）
 babel.transformFileSync('filename.js', options);
 // => { code, map, ast }
 
 // Babel AST转码
 babel.transformFromAst(ast, code, options);
 // => { code, map, ast }

```
配置对象options，可以参看官方文档http://babeljs.io/docs/usage/options/。

例子:


```
var es6Code = 'let x = n => n + 1';
 var es5Code = require('babel-core')
   .transform(es6Code, {
     presets: ['es2015']
   })
   .code;
 // '"use strict";\n\nvar x = function x(n) {\n  return n + 1;\n};'
```

## babel-polyfill VS babel-runtime VS babel-preset-env
### 不同点
#### 转换的代码不同
- babel-preset-env：转换语法，需要搭配polyfill使用
- babel-polyfill：

    - 转换promise等和内置对象原型上的方法，但是不能按需引入，一引就引全部

    但是可以通过env配置{useBuiltIns: 'usage'}来实现按需引入，但是对于原型链上的方法仅仅按照方法名去匹配

    - 会污染全局变量；
- babel-runtime：可以按需引入polyfill；防止babel帮助函数在多个文件中重复声明；防止全局变量污染

    - Babel < 7.4.0：不支持原型链上的方法，要同时设置corejs才能防止全局变量污染

    - Babel >= 7.4.0：设置corejs：3后支持原型链上的方法
## 总结
- 对于Babel < 7.4.0，<s>就直接升级到7.4.0吧</s>有以下方法转换es6、7代码：

    - polyfill + {useBuiltIns:usage}
    - @babel/runtime+corejs：2+polyfill

- Babel >= 7.4.0：

    = @babel/runtime+corejs：3
## demo
https://github.com/variinlkt/babel-demo
## 参考
https://juejin.im/post/5b07e79b51882538914a6039

https://www.w3ctech.com/topic/1708

https://juejin.im/post/5c09d6d35188256d9832df9d

https://www.jianshu.com/p/d078b5f3036a

https://juejin.im/post/5d0373a95188251e1b5ebb6c

http://www.ruanyifeng.com/blog/2016/01/babel.html

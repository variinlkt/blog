# NPM
自从当年抛弃jquery拥抱vue时，就开始接触了npm；平时用的npm命令不多，有时工作上遇到一些npm的不常用操作也没仔细想过为啥要这么做，于是今天就 <s>东拼西凑</s> 整理一篇npm文章来学习一下
## npm与yarn / cnpm
### cnpm
cnpm 使用软链接（也就是symlink/symbolic link/符号链接） 的安装方式，最大限度地提高了安装速度，生成的 node_modules 目录采用的是和 npm 不一样的布局

让我们来康康cnpm安装的目录结构（以安装redux为栗子）：

![](https://github.com/variinlkt/blog/blob/master/imgs/fullsizeoutput_5.jpeg)

这是npm的：

![](https://github.com/variinlkt/blog/blob/master/imgs/fullsizeoutput_6.jpeg)

里面的js-token、redux啥的都是软链接，链接指向了下划线开头的带版本号的文件夹

这样做的优点就是安装速度很快，因为采用了软连接的方式加上多线程请求【但是我不明白为啥软链接能加快请求速度，这里留个坑我还要去查下】，多个模块同时下载、解析、安装。

缺点就是目录结构跟npm安装的不一样，这可能会导致一些问题；以及不会生成lock文件。

### yarn
#### 速度快
- 并行安装：
无论 npm 还是 Yarn 在执行包的安装时，都会执行一系列任务。npm 是按照队列执行每个 package，也就是说必须要等到当前 package 安装完成之后，才能继续后面的安装。而 Yarn 是**同步执行所有任务**，提高了性能。
- 离线模式：
如果之前已经安装过一个软件包，用**Yarn再次安装时之间从缓存中获取**，就不用像npm那样再从网络下载了。
#### 安装版本统一
为了防止拉取到不同的版本，Yarn 有一个锁定文件 (lock file) 记录了被确切安装上的模块的版本号。每次只要新增了一个模块，Yarn 就会创建（或更新）yarn.lock 这个文件。这么做就保证了，每一次拉取同一个项目依赖时，使用的都是一样的模块版本。
## npm install

### 安装算法
https://docs.npmjs.com/cli/install#algorithm
> load the existing node_modules tree from disk
>
> clone the tree
>
> fetch the package.json and assorted metadata分类元数据 and add it to the clone
>
> walk the clone and add any missing dependencies
>   - dependencies will be added as close to the top as is possible
>   without breaking any other modules
>
> compare the original tree with the cloned tree and make a list of
> actions to take to convert one to the other
>
> execute all of the actions, deepest first
>   - kinds of actions are install, update, remove and move


```
对于这种package{dep}结构：A{B,C}, B{C}, C{D}，该算法产生：

A
+-- B
+-- C
+-- D
也就是说，A已经导致C安装在更高级别的事实满足了从B到C的依赖性。D仍然安装在顶层，因为没有任何冲突。

对于A{B,C}, B{C,D@1}, C{D@2}，这个算法产生：

A
+-- B
+-- C
   `-- D@2
+-- D@1
因为B的D @ 1 将安装在顶层，所以C现在必须 私下安装D @ 2 。该算法是确定性的，但是如果请求以不同顺序安装两个依赖性，则可以产生不同的树。
```
显而易见，多个模块依赖同一个模块时，**被依赖的模块会尽可能将包安装到更高的层级**。对于复杂的情况, npm 都会在安装时遍历整个依赖树，计算出最合理的文件夹安装方式，使得所有被重复依赖的包都可以去重安装。

这种扁平化的结构导致我们没有办法通过node_modules的目录结构来得知包的依赖项，想要查看 app 的直接依赖项：


```
npm ls --depth 1
```

#### npm i -g
与本地依赖包不同，如果我们通过 `npm install --global` 全局安装包到全局目录时，得到的目录依然是采用**简单的递归安装方法**得到的目录结构。

#### 局限性
npm flat-out拒绝安装 name@version已经存在于包文件夹祖先树中任何位置的任何内容。这可以用--force标志覆盖，但在大多数情况下可以通过更改本地包名称来解决。

这样做的原因是为了防止包之间的循环依赖
> A -> B -> A' -> B' -> A -> B -> A' -> B' -> A -> ...
>
>where A is some version of a package, and A' is a different version of the same package. Because B depends on a different version of A than the one that is already in the tree, it must install a separate copy. The same is true of A', which must install B'. Because B' depends on the original version of A, which has been overridden, the cycle falls into infinite regress.
#### 循环依赖
既然讲到了循环依赖，就顺便讲下node模块之间循环依赖的解法8
##### commonjs
- 加载时执行
- 已加载的模块会进行缓存，不会重复加载

因为模块进行了缓存，所以循环依赖没有得逞，但是最好的写法是在循环依赖的每个模块中**先写 exports 语句，再写 require 语句**，利用 CommonJS 的缓存机制，在 require() 其他模块之前先把自身要导出的内容导出，这样就能**保证其他模块在使用时可以取到正确的值**

##### es6
- import 静态执行，import 命令会被 JavaScript 引擎静态分析，优先于模块内的其他内容执行。
- export 动态绑定，export 命令输出的接口，与其对应的值是动态绑定关系，通过该接口可以实时取到模块内部的值。

因为 import 是在编译阶段执行的，这样就使得程序在编译时就能确定模块的依赖关系，一旦发现循环依赖，ES6 本身就不会再去执行依赖的那个模块了，所以程序可以正常结束。

虽然es6支持这种循环依赖，但是当它在发现有循环依赖时，会把import的循环依赖赋值为undefined，例如：

```
//foo.js
console.log('foo is running');
import {bar} from './bar'
console.log('bar = %j', bar);
setTimeout(() => console.log('bar = %j after 500 ms', bar), 500);
export let foo = false;
console.log('foo is finished');

//bar.js
console.log('bar is running');
import {foo} from './foo';
console.log('foo = %j', foo)
export let bar = false;
setTimeout(() => bar = true, 500);
console.log('bar is finished');

//执行 node foo.js 时会输出如下内容：
bar is running
foo = undefined
bar is finished
foo is running
bar = false
foo is finished
bar = true after 500 ms

```

## npm link
npm link用于给一个包创建一个软链接，链接到npm link执行该命令的包（哇这不就是cnpm用到的）

### 包链接过程:

```
npm link (in package dir)
```

- 为npm包目录创建软链接，将其链到`{prefix}/lib/node_modules/<package>`。
- 将包中的可执行文件bin链接到`{prefix}/bin/{name}`。


```
npm link [<@scope>/]<pkg>[@<version>]
```

- `npm link <package-name>`将创建从全局安装的`<package-name>`到`node_modules/ 当前文件夹`的符号链接。


具体用法：
```
cd ~/projects/node-redis    # go into the package directory
npm link                    # creates global link
cd ~/projects/node-bloggy   # go into some other package directory.
npm link redis              # link-install the package
```

### 作用
用于新开发npm模块在具体项目中进行试验，这样可以方便对模块进行测试，


## package.lock.json
package-lock.json 的作用是锁定依赖安装结构，如果查看这个 json 的结构，会发现与 node_modules 目录的文件层级结构是一一对应的。
### semver
https://semver.org/lang/zh-CN/

semver，即Semantic Versioning，语义化版本

#### 依赖地狱
指开发者安装某个软件包时，发现这个软件包里又依赖不同特定版本的其它软件包。随着系统功能越来越复杂，依赖的软件包越来越多，依赖关系也越来越深，这个时候可能面临版本控制被锁死的风险。

因此，Github 起草了一个具有指导意义的，统一的版本号表示规则，称为 Semantic Versioning(语义化版本表示)。该规则规定了版本号如何表示，如何增加，如何进行比较，不同的版本号意味着什么。



#### 版本格式
版本格式：主版本号.次版本号.修订号，版本号递增规则如下：

- 主版本号(major)：当你做了不兼容的 API 修改，
- 次版本号(minor)：当你做了向下兼容的功能性新增，可以理解为Feature版本，
- 修订号(patch)：当你做了向下兼容的问题修正，可以理解为Bug fix版本。
先行版本号及版本编译信息可以加到“主版本号.次版本号.修订号”的后面，作为延伸。
#### 依赖的版本范围
语义化版本范围规定：

- ~：只升级修订号
- ^：升级次版本号和修订号
- *：升级到最新版本

例如：

```
react@~16.0.1：>=react@16.0.1 && < react@16.1.0
redux@^3.7.2：>=redux@3.7.2 && < redux@4.0.0
lodash@*：lodash@latest
```


#### 先行版本
当要发布大版本或者核心的Feature时，但是又不能保证这个版本的功能 100% 正常。这个时候就需要通过发布先行版本。比较常见的先行版本包括：内测版、灰度版本了和RC版本。Semver规范中使用alpha、beta、rc来修饰即将要发布的版本。它们的含义是：

- alpha: 内部版本
- beta: 公测版本
- rc: 即Release candiate，正式版本的候选版本

比如：1.0.0-alpha.0, 1.0.0-alpha.1, 1.0.0-beta.0, 1.0.0-rc.0, 1.0.p-rc.1 等版本。alpha, beta, rc后需要带上次数信息。


#### npm包发布
通常我们发布一个包到npm仓库时，我们的做法是先修改 package.json 为某个版本，然后执行 npm publish 命令。手动修改版本号的做法建立在你对Semver规范特别熟悉的基础之上，否则可能会造成版本混乱。npm 考虑到了这点，它提供了相关的命令来让我们更好的遵从Semver规范：

升级补丁版本号：npm version patch
升级小版本号：npm version minor
升级大版本号：npm version major


## npm script
npm script它提供了一个简单的接口用来调用工程相关的脚本

package.json 中 scripts 字段可以定义脚本

### 原理
npm 脚本的原理非常简单。**每当执行npm run，就会自动新建一个 Shell，在这个 Shell 里面执行指定的脚本命令**。因此，只要是 Shell（一般是 Bash）可以运行的命令，就可以写在 npm 脚本里面。

比较特别的是，npm run新建的这个 Shell，**会将当前目录的node_modules/.bin子目录加入PATH变量，执行结束后，再将PATH变量恢复原样**。

这意味着，当前目录的node_modules/.bin子目录里面的所有脚本，都可以直接用脚本名调用，而不必加上路径。比如，当前项目的依赖里面有 Mocha，只要直接写mocha test就可以了。

```
"test": "mocha test"
```

由于 npm 脚本的唯一要求就是可以在 Shell 执行，因此它不一定是 Node 脚本，任何可执行文件都可以写在里面。

npm 脚本的退出码，也遵守 Shell 脚本规则。**如果退出码不是0，npm 就认为这个脚本执行失败。**
### 钩子函数
npm 脚本有pre和post两个钩子，例如：
用户执行npm run build的时候，会自动按照下面的顺序执行。

```
npm run prebuild && npm run build && npm run postbuild
```

因此，可以在这两个钩子里面，完成一些准备工作和清理工作。下面是一个例子。


```
"clean": "rimraf ./dist && mkdir dist",
"prebuild": "npm run clean",
"build": "cross-env NODE_ENV=production webpack"
```

### 环境变量：process.env 对象
运行时变量：在npm run 的脚本执行环境内，可以通过环境变量的方式获取许多运行时相关信息，以下都可以通过 `process.env `对象访问获得：

```
npm_lifecycle_event - 正在运行的脚本名称
npm_package_<key> - 获取当前包 package.json 中某个字段的配置值：如 npm_package_name 获取包名
npm_package_<key>_<sub-key> - package.json 中嵌套字段属性：如 npm_pacakge_dependencies_webpack 可以获取到 package.json 中的 dependencies.webpack 字段的值，即 webpack 的版本号
```



### 通配符&传参

*表示任意文件名，**表示任意一层子目录。

如果要将通配符传入原始命令，防止被 Shell 转义，要将星号转义。

传入参数，要使用--标明。
对于上面的脚本 "test": "mocha" 如果希望给 mocha 传入一些选项，比如希望执行：

mocha --reporter spec
需要这样执行 npm test：

npm test -- --reporter spec


### 执行顺序
- 并行执行（即同时的平行执行），可以使用&符号。
- 继发执行（即只有前一个任务成功，才执行下一个任务），可以使用&&符号。

### node_modules/.bin 

node_modules/.bin 目录，保存了依赖目录中所安装的可供调用的命令行包。
例如 webpack 就属于一个命令行包。如果我们在安装 webpack 时添加 --global 参数，就可以在终端直接输入 webpack 进行调用。但如果不加 --global 参数，我们会在 node_modules/.bin 目录里看到名为 webpack 的文件，如果在终端直接输入 ./node_modules/.bin/webpack 命令，一样可以执行。

这是因为 webpack 在 package.json 文件中定义了 bin 字段为:


```
{
    "bin": {
        "webpack": "./bin/webpack.js"
    }
}
```


npm 执行 install 时，会分析每个依赖包的 package.json 中的 bin 字段，并将其包含的条目安装到`./node_modules/.bin/<command>`中。

而如果是全局模式安装，则会在 npm 全局安装路径的 bin 目录下创建指向   `<file>` 名为` <command> `的软链。因此，`./node_modules/.bin/webpack` 文件在通过命令行调用时，实际上就是在执行 `node ./node_modules/.bin/webpack.js` 命令。

### script生命周期
还有一些 script 会在模块安装，发布，卸载等不同的生命周期被执行。

prepublish, publish, postpublish：发布模块

preinstall, install, postinstall：安装模块

preuninstall, uninstall, postuninstall：卸载模块

preversion, version, postversion：在使用 npm version 修改版本号的时候执行

pretest, test, posttest：执行 npm test 的时候

prestop, stop, poststop：执行 npm stop 的时候

prestart, start, poststart：执行 npm start 的时候

prerestart, restart, postrestart：执行 npm restart 的时候

这些 script 会在不同的时刻自动被执行，这也是为什么 npm run test 可以简写为 npm test 的原因了，在执行 npm test 的时候会以次执行 pretest、test 和 posttest，当然了，如果没有指定 pretest 和 posttest，会默默地跳过。

还有 npm restart 这个命令，它不单单执行 prerestart, restart, postrestart 这三个，而是按下面的顺序来执行：

prerestart

prestop

stop

poststop

restart

prestart

start

poststart

postrestart


### 编写 node 命令行工具
在 npm script 常常用到一些模块中的可执行程序，比如 eslint，webpack 等，那么要如何来自己编写一个命令行工具能，让它可以在 npm script 中被调用。

1. 编写命令行脚本
新建文件 cli.js，写入需要的逻辑。


```
console.log("hello");
```

2. 在 package.json 的 bin 字段中指定命令行文件名称和路径

```
{
    "bin": {
        "cli": "./cli.js"
    }
}
```

3. 指定解释器
当用户安装以后，通过 ./node_modules/.bin/cli 执行，会报错，原因是目前 shell 不知道使用什么解释器来执行这些代码，为此需要在脚本上方指定解释器。

```
#!usr/bin/env node
console.log("hello");
```

上面这一行在所有脚本文件中都可以看到，它叫做 SheBang 或者 HashBang，详见 Shebang_(Unix))，这行代码是告诉 shell 使用何种解释器来执行代码。usr/bin/env 是一个程序，usr/bin/env node 会找到当前 PATH 中的 node 来解释后面的代码。

有了这三步，就开发出了一个 node 的命令行工具。当用户安装这个模块的时候，npm 会在 node_modules/.bin/ 中创建文件，名为 package.json 中的 bin 指定的命令名，并链接至对应的脚本。此后就可以在 npm script 中使用它了。

多说两句，将上面的 #!usr/bin/env node 写入 JavaScript 文件第一行，不会报错。因为这是一个 UNIX 世界都认识的东西。通过 chmod +x cli.js，你可以使用 ./cli.js 直接执行它，因为这一行已经告诉 shell 使用 node 来执行它了。

## npm config
`npm config ls -l` 可查看 npm 的所有配置
修改配置的命令为 `npm config set <key> <value>`, 我们使用相关的常见重要配置:

- proxy, https-proxy: 指定 npm 使用的代理

```
npm config set proxy=http://127.0.0.1:8087
npm config delete proxy

npm config set https-proxy http://server:port
npm config delete https-proxy
```

- registry 指定 npm 下载安装包时的源，默认为 https://registry.npmjs.org/ 可以指定为私有 Registry 源

```
npm config set registry=http://registry.npmjs.org
```

- package-lock 指定是否默认生成 package-lock 文件，建议保持默认 true
save true/false 指定是否在 npm install 后保存包为 dependencies, npm 5 起默认为 true

删除指定的配置项命令为` npm config delete <key>`.

## .npmrc
npmrc 文件可以永久配置npm，避免手动config

这样的 npmrc 文件优先级由高到低包括：

- 工程内配置文件: /path/to/my/project/.npmrc
- 用户级配置文件: ~/.npmrc
- 全局配置文件: $PREFIX/etc/npmrc (即npm config get globalconfig 输出的路径)
- npm内置配置文件: /path/to/npm/npmrc

栗子：

```
//.npmrc
registry=http://registry.cnpmjs.org
```




## 参考文章
https://segmentfault.com/a/1190000007684156

http://www.ruanyifeng.com/blog/2016/10/npm_scripts.html

https://juejin.im/post/5ab3f77df265da2392364341#heading-15

https://segmentfault.com/a/1190000014405355

https://juejin.im/post/5a6008c2f265da3e5033cd93#heading-8

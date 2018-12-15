## Next.js
[官网地址](https://nextjs.org/docs#setup)

[中文文档](https://juejin.im/post/5b97b9af5188255c672e9564)

next是一个react SSR 的脚手架，不需要去配置页面，在根路径下的pages文件夹中新建文件/文件夹就能创建路由

next有自动化的code splitting，只会导入import中绑定以及被用到的代码

### css样式：
可以使用[styled-jsx](https://github.com/zeit/styled-jsx)来生成scoped css

也可以使用:

[@zeit/next-css](https://github.com/zeit/next-plugins/tree/master/packages/next-css)

[@zeit/next-sass](https://github.com/zeit/next-plugins/tree/master/packages/next-sass)

[@zeit/next-less](https://github.com/zeit/next-plugins/tree/master/packages/next-less)

注意： webpack方法将被执行两次，一次在服务端一次在客户端。你可以用isServer属性区分客户端和服务端来配置

有个配置文件next.config.js可以进行配置：


```
//next.config.js
const withCSS = require('@zeit/next-css')
const withSass = require('@zeit/next-sass')
const withLess = require('@zeit/next-less')

module.exports = withCSS(withSass(withLess({
    //支持css文件中引入图像文件
  cssLoaderOptions:{
    url: true,
  },

  webpack(config, {}) {
    config.module.rules.push({
      test: /\.(eot|woff|woff2|ttf|svg|png|jpg|gif)$/,
      use: {
        loader: 'url-loader',//只有url-loader能生效，这里写file-loader无法加载图像文件
        options: {
          limit: 100000
        }
      }
    })
    return config
  }
})))

```

webpack的第二个参数是个对象，你可以自定义配置它，对象属性如下所示：

- buildId - 字符串类型，构建的唯一标示
- dev - Boolean型，判断你是否在开发环境下
- isServer - Boolean 型，为true使用在服务端, 为false使用在客户端.
- defaultLoaders - 对象型 ，内部加载器, 你可以如下配置

- babel - 对象型，配置babel-loader.
- hotSelfAccept - 对象型， hot-self-accept-loader配置选项.这个加载器只能用于高阶案例。如 @zeit/next-typescript添加顶层 typescript 页面。


虽说next-css能解决大部分css import的问题，但是我遇到的坑是：
- npm 安装组件->组件中require css->报错
- npm 安装组件->组件中require另一个npm安装的组件->另一个npm安装的组件中require css->报错

第一个问题还是比较好解决的，其实一开始问题出现的场景是：

我写了一个组件，打包的时候没用babel编译->组件里面的导入模块都是import而不是require->通过npm安装组件->疯狂报错

![](https://dfiles.tita.com/Portal/110006/9a454f24796a4c7590f959e8cc15f4e3.png)
`Unhandled Rejection(SyntaxError):Unexpected identifier`

#### 为什么node_modules下的import会报错呢？

webpack4以上不是应该支持import语法吗？

因为next.js会对pages文件夹下的文件进行babel编译，但是**next.js阻止了对node_modules下的文件的es6import语法的编译**


当我们通过相对路径来引入node_modules下的文件的时候是可以的

#### 解决方法：使用 `babel-plugin-root-import`插件解决路径问题

引入 node_modules 中的模块如果需要通过babel/webpack编译，使用~关键字：

在.babelrc文件下：
![](https://dfiles.tita.com/Portal/110006/c58e533843174fa2af95f7b07c4c034c.png)

**对于node_modules文件夹下的组件require css报错也可以用该方法解决**

第二个问题在next-css的GitHub下的issue有很多人提到，但是都没有一个比较好的解决方法，个人的解决方法是直接将node_modules下组件中require的组件直接copy到组件里面了= =

### 重写head标签

写一个head组件：
```
import Head from 'next/head'

export default () => (
  <div>
    <Head>
      <title>My page title</title>
      <meta name="viewport" content="initial-scale=1.0, width=device-width" key="viewport" />
    </Head>
    <Head>
      <meta name="viewport" content="initial-scale=1.2, width=device-width" key="viewport" />
    </Head>
    <p>Hello world!</p>
  </div>
)
```
用key来避免重复渲染head标签

再在页面js中import

### getInitialProps

用来异步获取js普通对象，并绑定在props上，返回值应该是一个对象，然后在render函数中可以通过this.props来获取返回值：


```
import React from 'react'

export default class extends React.Component {
  static async getInitialProps({ req }) {
    const userAgent = req ? req.headers['user-agent'] : navigator.userAgent
    return { userAgent }
  }

  render() {
    return (
      <div>
        Hello World {this.props.userAgent}
      </div>
    )
  }
}

```

getInitialProps将不能使用在子组件中。只能使用在pages页面中

getInitialProps入参对象的属性如下：

- pathname - URL 的 path 部分
- query - URL 的 query 部分，并被解析成对象
- asPath - 显示在浏览器中的实际路径（包含查询部分），为String类型
- req - HTTP 请求对象 (只有服务器端有)
- res - HTTP 返回对象 (只有服务器端有)
- jsonPageRes - 获取数据响应对象 (只有客户端有)
- err - 渲染过程中的任何错误



注意：当页面初始化加载时，getInitialProps只会加载在**服务端**。

**只有当路由跳转（Link组件跳转或 API 方法跳转）时，客户端才会执行getInitialProps。**

### 路由

#### <Link>

```
// pages/index.js
import Link from 'next/link'

export default () =>
  <div>
    Click{' '}
    <Link href={{ pathname: '/about', query: { name: 'Zeit' }}} >
      <a>here</a>
    </Link>{' '}
    to read more
  </div>

```
在link标签中添加replace属性替换路由，passHref属性可以强制将href传递给子元素，scroll={false}属性禁止滚动到页面顶部

link标签支持onClick事件

#### next/router


```
import Router from 'next/router'

const handler = () =>
  Router.push({
    pathname: '/about',
    query: { name: 'Zeit' }
  })

export default () =>
  <div>
    Click <span onClick={handler}>here</span> to read more
  </div>


```
Router对象的 API 如下：

- route - 当前路由的String类型
- pathname - 不包含查询内容的当前路径，为String类型
- query - 查询内容，被解析成Object类型. 默认为{}
- asPath - 展现在浏览器上的实际路径，包含查询内容，为String类型
- push(url, as=url) - 页面渲染第一个参数 url 的页面，浏览器栏显示的是第二个参数 url
- replace(url, as=url) - performs a replaceState call with the given url
- beforePopState(cb=function) - 在路由器处理事件之前拦截.

这里的url对象可以通过输出this.props来看到：

![](https://github.com/variinlkt/blog/blob/master/imgs/1B3EC05EA5291ED64C294C2E37F193AA.png?raw=true)

#### 监听路由相关事件：

- routeChangeStart(url) - 路由开始切换时触发
- routeChangeComplete(url) - 完成路由切换时触发
- routeChangeError(err, url) - 路由切换报错时触发
- beforeHistoryChange(url) - 浏览器 history 模式开始切换时触发
- hashChangeStart(url) - 开始切换 hash 值但是没有切换页面路由时触发
- hashChangeComplete(url) - 完成切换 hash 值但是没有切换页面路由时触发

#### 使用：

```
const handleRouteChange = url => {
  console.log('App is changing to: ', url)
}
//绑定监听
Router.events.on('routeChangeStart', handleRouteChange)
//取消监听
Router.events.off('routeChangeStart', handleRouteChange)

Router.events.on('routeChangeError', (err, url) => {
  if (err.cancelled) {//路由加载被取消
    console.log(`Route to ${url} was cancelled!`)
  }
})

```

#### shallow：true
用于不想触发getInitialProps生命周期的场景：

```
// Current URL is "/"
const href = '/?counter=10'
const as = href
Router.push(href, as, { shallow: true })

```


我用到的是router，遇到了一些小坑：

场景：

从a页跳转到b页->b页成功加载->刷新b页->报错Router.query不存在

出现这种情况的原因是**我把Router.query写到了getInitialProps函数里面**

前面提到：
当页面初始化加载时，getInitialProps只会加载在服务端，只有当路由跳转（Link组件跳转或 API 方法跳转）时，客户端才会执行getInitialProps。

Router.query是从浏览器url中取的数据，如果写在getInitialProps函数中，通过跳转进入当前页面是ok的，但是一旦用户刷新当前页面就会报错

正确的做法是写在componentDidMount中

### 获取后端数据
如果是在getInitialProps函数中获取的话：

```
import fetch from 'isomorphic-unfetch'

static async getInitialProps({ query }){ 

  const { name, version } = query 

  const res = await fetch(url) 

  const detail = await res.json() return { detail }

}
```
要提一句，这里的fetch只能设置一个绝对地址，可以通过以下方法写个相对地址：


```
const baseUrl = req ? `${req.protocol}://${req.get('Host')}` : '';

await fetch（baseUrl+query）
```

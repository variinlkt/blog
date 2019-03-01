# 关于react-router-dom的学习

## 与react-router的区别
- react-router: 实现了路由的核心功能
- react-router-dom: 基于react-router，加入了在浏览器运行环境下的一些功能，例如：Link组件

**react-router-dom依赖react-router，所以我们使用npm安装依赖的时候，只需要安装相应环境下的库即可，不用再显式安装react-router。**
基于浏览器环境的开发，只需要安装react-router-dom；基于react-native环境的开发，只需要安装react-router-native。npm会自动解析react-router-dom包中package.json的依赖并安装。

## React-router-dom 中的组件
其实就是vue-router的hash和history实现前端路由
### `<HashRouter>`
hashrouter是以#号方式匹配路由，从url中可以看出来，这个地址对于后端来说，全部指向同一个地址；原理是hash。（hashchange事件）

```
import { HashRouter } from 'react-router-dom';

<HashRouter>
  <App />
</HashRouter>
```
- hashType: string
window.location.hash 使用的 hash 类型，有如下几种：

    - slash - 后面跟一个斜杠，例如 #/ 和 #/sunshine/lollipops
    - noslash - 后面没有斜杠，例如 # 和 #sunshine/lollipops
    - hashbang - Google 风格的 ajax crawlable，例如 #!/ 和 #!/sunshine/lollipops

默认为 slash。
- basename: string
- getUserConfirmation: func
- children: node
要呈现的单个子元素（组件）。
### `<BrowserRouter>`
而browserrouter原理是history，不同的路由对于后端也是不同的地址。(pushState, replaceState 和 popstate 事件) 

```
import { BrowserRouter } from 'react-router-dom';

<BrowserRouter
  basename={string}
  forceRefresh={bool}
  getUserConfirmation={func}
  keyLength={number}
>
  <App />
</BrowserRouter>
```
- basename=基准url
```
<BrowserRouter basename="/calendar">
  <Link to="/today" />
</BrowserRouter>
```
最终转换为：

```
<a href="/calendar/today" />
```
- forceRefresh=强制刷新，在不支持history api的浏览器中使用此功能

- getUserConfirmation=用于确认导航的函数
```
// 这是默认的确认函数
const getConfirmation = (message, callback) => {
  const allowTransition = window.confirm(message);
  callback(allowTransition);
}

<BrowserRouter getUserConfirmation={getConfirmation} />
```
- keyLength: number
location.key 的长度，默认为 6。

### `<Link>`

```
import { Link } from 'react-router-dom';

<Link to="/about">About</Link>
```
- to: string
一个字符串形式的链接地址，通过 pathname、search 和 hash 属性创建。
- to: object
一个对象形式的链接地址，可以具有以下任何属性：

    - pathname - 要链接到的路径
    - search - 查询参数
    - hash - URL 中的 hash，例如 #the-hash
    - state - 存储到 location 中的额外状态数据
    
- replace: bool
当设置为 true 时，点击链接后将替换历史堆栈中的当前条目，而不是添加新条目。默认为 false。
- innerRef: func
允许访问组件的底层引用。


```
const refCallback = node => {
  // node 指向最终挂载的 DOM 元素，在卸载时为 null
}

<Link to="/" innerRef={refCallback} />
```
### `<NavLink>`

```
const activeStyle = {
  fontWeight: 'bold',
  color: 'red'
};
const oddEvent = (match, location) => {
  if (!match) {
    return false;
  }
  const eventID = parseInt(match.params.eventID);
  return !isNaN(eventID) && eventID % 2 === 1;
}
<NavLink to="/faq" 
    activeClassName="selected"
    activeStyle={activeStyle}
    isActive={oddEvent}
>
    FAQs
</NavLink>
```
- activeClassName: string
当元素处于激活状态时应用的类，默认为 active。它将与 className 属性一起使用。
- activeStyle: object
当元素处于激活状态时应用的样式。
- exact: bool
如果为 true，则只有在位置完全匹配时才应用激活类/样式。
- strict: bool
如果为 true，则在确定位置是否与当前 URL 匹配时，将考虑位置的路径名后面的斜杠
- isActive: func
添加额外逻辑以确定链接是否处于激活状态的函数。如果你要做的不仅仅是验证链接的路径名与当前 URL 的路径名相匹配，那么应该使用它。

### `<Prompt>`
用于在位置跳转之前给予用户一些确认信息。当你的应用程序进入一个应该阻止用户导航的状态时（比如表单只填写了一半），弹出一个提示。

```
import { Prompt } from 'react-router-dom';

<Prompt
  when={formIsHalfFilledOut}
  message="你确定要离开当前页面吗？"
/>
```
- message: func|string

将在用户试图导航到下一个位置时调用。需要返回一个字符串以向用户显示提示，或者返回 true 以允许直接跳转。

```
<Prompt message={location => {
  const isApp = location.pathname.startsWith('/app');

  return isApp ? `你确定要跳转到${location.pathname}吗？` : true;
}} />
```
- when: bool
在应用程序中，你可以始终渲染` <Prompt> `组件，并通过设置 when={true} 或 when={false} 以阻止或允许相应的导航，而不是根据某些条件来决定是否渲染 `<Prompt>` 组件。

### `<MemoryRouter>`
将 URL 的历史记录保存在内存中的 <Router>（不读取或写入地址栏）。在测试和非浏览器环境中很有用，例如 React Native。
- initialEntries: array
历史堆栈中的一系列位置信息。这些可能是带有 {pathname, search, hash, state} 的完整位置对象或简单的字符串 URL。
- initialIndex: number
initialEntries 数组中的初始位置索引。
- getUserConfirmation: func
用于确认导航的函数。当 `<MemoryRouter> `直接与` <Prompt> `一起使用时，你必须使用此选项。

```
<MemoryRouter
  initialEntries={[ '/one', '/two', { pathname: '/three' } ]}
  initialIndex={1}
>
  <App/>
</MemoryRouter>
```
### `<Redirect>`
使用` <Redirect> `会导航到一个新的位置。新的位置将覆盖历史堆栈中的当前条目，例如服务器端重定向（HTTP 3xx）。


```
import { Route, Redirect } from 'react-router-dom';

<Route exact path="/" render={() => (
  loggedIn ? (
    <Redirect to="/dashboard" />
  ) : (
    <PublicHomePage />
  )
)} />
```
- push: bool
如果为 true，重定向会将新的位置推入历史记录，而不是替换当前条目。
- to: string|object
### `<Route>`
它最基本的职责是在其 path 属性与某个 location 匹配时呈现一些 UI。

```
import { BrowserRouter as Router, Route } from 'react-router-dom';

<Router>
  <div>
    <Route exact path="/" component={Home} />
    <Route path="/news" component={News} />
  </div>
</Router>
```
如果应用程序的位置是 /，那么 UI 的层次结构将会是：

```
<div>
  <Home />
  <!-- react-empty: 2 -->
</div>
```
**路由始终在技术上被“渲染”，即使它的渲染为空。**

#### `<Route>` 渲染方式：

- `<Route component>`指定只有当位置匹配时才会渲染的 React 组件，该组件会接收 route props 作为属性。


```
const User = ({ match }) => {
  return <h1>Hello {match.params.username}!</h1>
}
<Route path="/user/:username" component={User} />
```
当你使用 component时，Router 将根据指定的组件，使用 **React.createElement 创建一个新的 React 元素**。这意味着，**如果你向 component 提供一个内联函数，那么每次渲染都会创建一个新组件**。**这将导致现有组件的卸载和新组件的安装，而不是仅仅更新现有组件**。当使用内联函数进行内联渲染时，请使用 render 或 children
- `<Route render>`
使用 render 可以方便地进行内联渲染和包装，而无需进行上文解释的不必要的组件重装。

你可以传入一个函数，以在位置匹配时调用，而不是使用 component 创建一个新的 React 元素

```
// 方便的内联渲染
<Route path="/home" render={() => <div>Home</div>} />

// 包装
const FadingRoute = ({ component: Component, ...rest }) => (
  <Route {...rest} render={props => (
    <FadeIn>
      <Component {...props} />
    </FadeIn>
  )} />
)

<FadingRoute path="/cool" component={Something} />
```

**`<Route component> `优先于 `<Route render>`，因此不要在同一个 `<Route>` 中同时使用两者**
- `<Route children>`
有时候不论 path 是否匹配位置，你都想渲染一些内容。在这种情况下，你可以使用 children 属性。除了不论是否匹配它都会被调用以外，它的工作原理与 render 完全一样。
children 渲染方式接收所有与 component 和 render 方式相同的 route props，除非路由与 URL 不匹配，不匹配时 match 为 null。这允许你可以根据路由是否匹配动态地调整用户界面。如下所示，如果路由匹配，我们将添加一个激活类：

```
const ListItemLink = ({ to, ...rest }) => (
  <Route path={to} children={({ match }) => (
    <li className={match ? 'active' : ''}>
      <Link to={to} {...rest} />
    </li>
  )} />
)

<ul>
  <ListItemLink to="/somewhere" />
  <ListItemLink to="/somewhere-else" />
</ul>
```

这对动画也很有用：

```
<Route children={({ match, ...rest }) => (
  {/* Animate 将始终渲染，因此你可以利用生命周期来为其子元素添加进出动画 */}
  <Animate>
    {match && <Something {...rest} />}
  </Animate>
)} />
```

**`<Route component> `和 `<Route render>` 优先于 `<Route children>`，因此不要在同一个 `<Route> `中同时使用多个。**
#### Route props
三种渲染方式都将提供相同的三个路由属性：

- match
- location
- history

### `<Router>`
所有 Router 组件的通用低阶接口。通常情况下，应用程序只会使用其中一个高阶 Router：

- `<BrowserRouter>`
- `<HashRouter>`
- `<MemoryRouter>`
- `<NativeRouter>`
- `<StaticRouter>`
### `<Switch>`
用于渲染与路径匹配的第一个子 `<Route>` 或 `<Redirect>`。

#### 与 `<Route>` 有何不同？

**`<Switch>` 只会渲染一个路由**。相反，仅仅定义一系列 `<Route>` 时，每一个与路径匹配的 `<Route> `都将包含在渲染范围内。

```
<Route path="/about" component={About} />
<Route path="/:user" component={User} />
<Route component={NoMatch} />
```

## 参考文章
https://segmentfault.com/a/1190000014294604#articleHeader7

https://github.com/mrdulin/blog/issues/42

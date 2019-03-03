# 关于React Context

如果一个组件设置了context，那么它的子组件都可以直接访问到里面的内容，它就像这个组件为根的子树的全局变量。

任意深度的子组件都可以通过 contextTypes 来声明你想要的 context 里面的哪些状态，然后可以通过 this.context 访问到那些状态。

context 打破了组件和组件之间通过 props 传递数据的规范，极大地增强了组件之间的耦合性。而且，就如全局变量一样，context 里面的数据能被随意接触就能被随意修改，每个组件都能够改 context 里面的内容会导致程序的运行不可预料。

但是这种机制对于前端应用状态管理来说是很有帮助的，因为毕竟很多状态都会在组件之间进行共享，

context会给我们带来很大的方便。一些第三方的前端应用状态管理的库（例如 Redux）就是充分地利用了这种机制给我们提供便利的状态管理服务。
## 使用
父级设置context，子级读取context，他们分别需要定义childContextTypes和contextTypes来定义变量类型
父组件的实现：

```
const PropTypes = require('prop-types');
class MessageList extends React.Component {
  getChildContext() {
    return {color: "purple"};
  } // 返回childContext具体值

  render() {
    const children = this.props.messages.map((message) =>
      <Message text={message.text} />
    );
    return <div>{children}</div>;
  }
}

MessageList.childContextTypes = {  // 通过childContextTypes定义context中变量类型
  color: PropTypes.string
};
```



子组件获取父组件设置的context：

```
const PropTypes = require('prop-types');
class Button extends React.Component {
  render() {
    return (
      <button style={{background: this.context.color}}> // 使用context
        {this.props.children}
      </button>
    );
  }
}

Button.contextTypes = { // 定义context
 变量类型 color: PropTypes.string
};
```

子级通过contextTypes来标识自己接收context，然后就能直接使用context了。

如果子级没有定义contextTypes，则context对象将为空。另外，如果定义了contextTypes，以下回调中都会添加一个新参数context参数：
- constructor(props, context)
- componentWillReceiveProps(nextProps, nextContext)
- shouldComponentUpdate(nextProps, nextState, nextContext)
- componentWillUpdate(nextProps, nextState, nextContext)
- componentDidUpdate(prevProps, prevState, prevContext)

最后，函数式组件中，通过定义contextType也能使用context：

```
const PropTypes = require('prop-types');

const Button = ({children}, context) =>
  <button style={{background: context.color}}>
    {children}
  </button>;

Button.contextTypes = {color: PropTypes.string};
```
## 对比vue
vue如果要实现隔级组件通信或兄弟组件通信，可以先实现一个bus组件：

```
import Vue from 'vue';
export default Bus
```
然后父子组件分别引入这个bus组件

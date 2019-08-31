# React hooks
react hooks 是react 16.8引入的新特性
## 为什么要用react hooks
### 业务逻辑 vs 生命周期 + 业务逻辑
我们都知道react有两种类型的组件，一种是class，一种是函数式组件，函数式组件是一个function，在function内部是没有生命周期的，也没有this；而在class组件中存在生命周期和this；

当我们需要在组件的某些特定时间做一些业务逻辑处理时，需要在对应的生命周期中处理逻辑；而react hooks只能用于函数式组件，它能让用户专注于业务逻辑而无需分心思考业务逻辑需要放在哪个生命周期函数中。同时也极大地减少了代码长度。
### 没有this
为什么没有this也是优点呢？请看以下代码，function component和class component有什么区别呢？

梳理一下逻辑，用户点击按钮，三秒后会弹出一个弹框，输出传入该组件的user
当传入的props在这三秒中发生了变化，function component和class component的输出会有什么变化呢？

答案就是：function component会输出点击时传入的user值，class component会输出变化后的user值。

虽然这样的场景比较少，但是一旦发生了可能会产生意料之外的结果，这也是为啥 react 团队推荐使用hooks的原因之一。
#### 为什么删除了will生命周期函数
react 16.8中删除了componentwillmount等will生命周期函数。原因是：v17中将推出的async render和react fiber架构会导致生命周期可以被打断，像componentwillmount等生命周期函数可能会被执行多次；而componentwillmount这个生命函数的命名听起来像是只会被调用一次，使用者可能会产生误解。

### 提高业务逻辑复用性
这里也顺便介绍下其他代码复用的几种方案及其各自的利弊：
#### mixin
Mixin其实就是混入的方式，将功能注入到原型对象中，来实现代码和逻辑的复用。但是副作用也是很明显的，他会对原有的对象造成污染，函数来源不确定，并且可能出现命名冲突的问题。
#### class继承
#### renderprops
Render Props是一个类型为函数的props，来实现组件之间的代码复用。
```

import React from 'react'
import ReactDOM from 'react-dom'
import PropTypes from 'prop-types'
// 与 HOC 不同，我们可以使用具有 render prop 的普通组件来共享代码
class Mouse extends React.Component {
  static propTypes = {
    render: PropTypes.func.isRequired
  }
  state = { x: 0, y: 0 }
  handleMouseMove = (event) => {
    this.setState({
      x: event.clientX,
      y: event.clientY
    })
  }
  render() {
    return (
      <div style={{ height: '100%' }} onMouseMove={this.handleMouseMove}>
        {this.props.render(this.state)}
      </div>
    )
  }
}
const App = React.createClass({
  renderTitle =({x, y})=>{
	return 
      // render prop 给了我们所需要的 state 来渲染我们想要的
      <h1>The mouse position is ({x}, {y})</h1>
  }
  render() {
    return (
      <div style={{ height: '100%' }}>
        <Mouse render={this.renderTitle}/>
      </div>
    )
  }
})

```
Mouse 组件接受一个类型为函数的render props，然后在render中调用该方法，并将自身的state作为参数，这样，其他组件就能到Mouse的状态，并渲染想要的内容。并且我们注意到，组合是发生在render中的，也就是说在运行态下的动态组合，我们就可以用到原组件内部的任何state 或 props数据，以及生命周期函数等等。

#### Hoc：缺点：多层嵌套
所谓HOC，就是指高阶组件，他的概念类似于高阶函数，只是这里接受一个组件作为参数，经过装饰之后，返回一个新的组件。装饰的过程，就是将可复用的逻辑，附加到原有的组件上，可以是组件结构的扩展，也可以是功能的扩充。

## hooks原理
闭包
```
// 使用闭包来实现 useState 的简单逻辑:

const React = (function() {
  let _val

  return {
    useState(initialValue) {
      _val = _val || initialValue

      function setVal(value) {
        _val = value
      }

      return [_val, setVal]
    }
  }
})()

// useEffect

var React = (function() {
  let _val, _deps

  return {
    useState(initialValue) {
      _val = _val || initialValue

      function setVal(value) {
        _val = value
      }

      return [_val, setVal]
    },
    useEffect(callback, deps) {
      const ifUpdate = !deps

      // 判断 Deps 中的依赖是否改变
      const ifDepsChange = _deps ? !_deps.every((r, index) => r === deps[index]) : true

      if (ifUpdate || ifDepsChange) {
        callback()

        _deps = deps || []
      }
    }
  }
})()

```

## hooks原则及api
### 原则
1、只能在React函数中调用Hooks 

- React 函数组件
- 自定义hooks
2、只在最顶层使用Hooks，保证一个组件在每一次渲染的过程中，hooks都是按照完全相同的顺序执行

### api
https://react.docschina.org/docs/hooks-reference.html

## 参考文章
https://www.cnblogs.com/MuYunyun/p/10852005.html
https://overreacted.io/zh-hans/a-complete-guide-to-useeffect/
https://blog.csdn.net/neoveee/article/details/87619175

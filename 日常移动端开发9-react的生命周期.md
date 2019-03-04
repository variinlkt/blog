# react的生命周期
## react v17前的生命周期
![](https://github.com/variinlkt/blog/blob/master/imgs/life_cycle.png)

生命周期可以被打断，一旦生命周期被打断，下次恢复的时候又会再跑一次生命周期

因此componentWillMount，componentWillReceiveProps， componentWillUpdate都不能保证只在挂载/拿到props/状态变化的时候刷新一次了，所以这三个方法被标记为不安全。
### 总结
#### 初始化时

##### getDefaultProps()
设置默认的props，es6中用 static defaultProps={} 设置组件的默认属性。在整个生命周期只执行一次。

##### getInitialState()
在使用es6的class语法时是没有这个钩子函数的，可以直接在constructor中定义this.state。此时可以访问this.props。

##### componentWillMount() 
ajax数据的拉取操作，定时器的启动。
组件初始化时调用，以后组件更新不调用，整个生命周期只调用一次，此时可以修改state。

##### render()
React最重要的步骤，创建虚拟dom，进行diff算法，更新dom树都在此进行。此时就不能更改state了。

##### componentDidMount() 动画的启动，输入框自动聚焦
组件渲染之后调用，可以通过this.getDOMNode()获取和操作dom节点，只调用一次。

#### 更新时

##### componentWillReceivePorps(nextProps)
组件初始化时不调用，组件接受新的props时调用。不管父组件传递给子组件的props有没有改变，都会触发。

##### shouldComponentUpdate(nextProps, nextState)
React性能优化非常重要的一环。组件接受新的state或者props时调用，我们可以设置在此对比前后两个props和state是否相同，如果相同则返回false阻止更新，因为相同的属性状态一定会生成相同的dom树，这样就不需要创造新的dom树和旧的dom树进行diff算法对比，节省大量性能，尤其是在dom结构复杂的时候。不过**调用this.forceUpdate会跳过此步骤。**

##### componentWillUpdate(nextProps, nextState)
组件初始化时不调用，只有在组件将要更新时才调用，此时可以修改state

##### render()

##### componentDidUpdate()
组件初始化时不调用，组件更新完成后调用，此时可以获取dom节点。

#### 卸载时

##### componentWillUnmount() 定时器的清除
组件将要卸载时调用，一些事件监听和定时器需要在此时清除。



## react v17的生命周期
![](http://imweb-io-1251594266.file.myqcloud.com/FpthV14jUJPChMJI8BQtuITotlMt)

### 1、mounting过程：一个组件被实例化创建并且插入到dom中

调用方法：

#### constructor()：初始化state（只能使用this.state来初始化），在组件实例绑定事件handler

触发时间：在组件被mount前被调用

用法：初始化state



#### static getDerivedStateFromProps(nextProps,prevState)

触发时间：组件每次被rerender的时候，包括在组件构建之后(虚拟dom之后，实际dom挂载之前)，和每次获取新的props或state之后

用法：根据props更新state、传入新的props时重新异步取数据（getDerivedStateFromProps+ componentDidUpdate）

```
// new

  static getDerivedStateFromProps(nextProps, prevState) {

    // Store prevId in state so we can compare when props change.

    if (nextProps.id !== prevState.prevId) {

      return {

        externalData: null,

        prevId: nextProps.id,

      };

    }

    // No state update necessary

    return null;

  }

  componentDidUpdate(prevProps, prevState) {

    if (this.state.externalData === null) {

      this._loadAsyncData(this.props.id);

    }

  }
```

每次接收新的props之后都会返回一个对象作为新的state，返回null则说明不需要更新state.

在组件实例化时代替了componentWillMount()

接收到新props时，代替了componentWillReceiveProps()和componentWillUpdate()

**由于父组件的原因重新渲染时这个方法会被调用，但是this.setState()不会重新调用这个方法**

===========

Q：为什么react不给一个prevProps参数？

A：prevProps第一次被调用的时候是null，每次更新都要判断耗性能

================

#### componentWillMount()/UNSAFE_componentWillMount

触发时间：在mounting发生前被调用，服务器端和客户端都只调用一次

在这个方法中调用setState不会导致额外的渲染，因为它在render前被调用

一般用来异步获取数据，但是**在新版生命周期里面不建议这么做**，因为：

1、会阻碍组件的实例化、渲染

2、调用setState不会导致额外的渲染

3、在v16.3之后react开始异步渲染,在异步渲染模式下,使用componentWillMount会被多次调用,并且存在内存泄漏等问题



在v17中该方法将被UNSAFE_componentWillMount取代

官方推荐使用getDerivedStateFromProps()

UNSAFE_componentWillMount是用来向下兼容的



#### render()

一个组件中必须有这个方法

如果shouldComponentUpdate()返回值为false不调用该方法

返回以下值之一：

react element

string&numbers

protals

null

booleans



#### componentDidMount()

触发时间：第一次插入dom完毕，即组件已经挂载到dom上了，在同步/异步模式下都仅触发一次

这个方法在组件被mount后立即调用

用法：请求数据、事件监听

在这个方法调用setState会导致一次额外的渲染，但是渲染会发生在浏览器更新屏幕之前，因此即使渲染了两次用户也看不到中间的状态，因此在用户的视觉体验上没有影响，但是会导致性能方面的问题

===========

Q：请求数据放在componentDidMount里面会不会比componentWillMount慢很多？

A：不会，因为render在willMount之后几乎是马上就被调用，根本等不到数据回来，同样需要render一次“加载中”的空数据状态，所以在didMount去取数据几乎不会产生影响。

============

##### componentWillMount和componentDidMount的区别：

- componentWillMount可以在**服务端**被调用，也可以在浏览器端被调用；而componentDidMount**只能在浏览器端**被调用。

- **在componentDidMount 被调用后,componentWillUnmount 一定会随后被调用到**；**componentWillMount可以被打断或调用多次，因此无法保证事件监听能在unmount的时候被成功卸载**，可能会引起内存泄露



### 2、updating：组件被插入到dom中，渲染完毕，props/state发生改变时触发的，此时组件会被重新渲染

#### componentWillReceiveProps(nextProps)/UNSAFE_componentWillReceiveProps()

如果组件自身的某个 state 跟其 props 密切相关的话，需要在 componentWillReceiveProps 中判断前后两个 props 是否相同,如果不同再将新的 props 更新到相应的 state 上去. 

**这样做一来会破坏 state 数据的单一数据源,导致组件状态变得不可预测,也会增加组件的重绘次数**



#### static getDerivedStateFromProps()

上面写了



#### shouldComponentUpdate(nextProps,nextState)

返回一个布尔值，返回false会停止更新过程，不会触发后续的渲染

shouldComponentUpdate返回true，接下来会依次调用componentWillUpdate、render和componentDidUpdate函数。



#### componentWillUpdate(nextProps,nextState) / UNSAFE_componentWillUpdate()

在接收到新props/state前立刻调用



#### render()

上面写了



#### getSnapshotBeforeUpdate(prevProps,prevState)

触发时间：update发生时，render之后，mount之前

**用法：在更新前记录原来的dom节点属性**



返回值作为componentDidUpdate的第三个参数



#### componentDidUpdate()

用法：在生命周期中由于state的变化触发请求、props更改引发的可视变化、传入新的props时重新异步取数据（getDerivedStateFromProps+ componentDidUpdate）

componentWillUpdate, componentWillReceiveProps在一次更新中可能会被触发多次，因此这种只希望触发一次的副作用应该放在保证只触发一次的componentDidUpdate中。



### 3、unmounting：组件要被移出dom

#### componentWillUnmount()

### 4、error handling：构造函数、生命周期或渲染出现错误时

#### componentDidCatch()

=======================

写文章好无聊啊，不如我们来复习一下vue的生命周期吧
## vue的生命周期
![](https://cn.vuejs.org/images/lifecycle.png)
### beforeCreate
el 和 data 并未初始化 
- 实例初始化
- 数据观测
- 暴露属性和方法

可以在这里加loading态
### created
完成了 data 数据的初始化，el没有
- 数据已绑定/this.$el未绑定
- 是否有el属性？是否有$mount()？都无则没有mounted/beforeMount钩子
- 是否有template属性？有则直接用内部的，调用render()渲染，无则调用外部的html，都没有则报错（render函数>内部template属性优先级>外部html）
- 编译模板
 	
在这结束loading

### beforeMount
完成了 el 和 data 初始化 
- 用编译好的html内容替换el指向的dom对象
- 创建vue实例的$el，并用它代替el属性

即：beforeMount时this.$el只是个div，mounted时this.$el的div里面有id和div内部绑定的数据
### mounted
完成挂载，可以在这里发起ajax请求
### beforeUpdate
- beforeUpdate调用后数据就会更新
- 数据改变->虚拟dom改变->调用这两个生命钩子改变视图
- 只有当数据和模板中的数据绑定了才会更新
### updated
### beforeDestroy
### destroyed
- 解绑/销毁

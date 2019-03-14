# redux和react-redux
Redux是一种架构模式，是由flux发展而来的。
![](https://img-blog.csdn.net/20180715103243690?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTEyMTU2Njk=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)
## redux三大原则
1. 唯一数据源
2. 状态只读
3. 数据改变只能通过纯函数（reducer）完成

## Redux核心
Redux主要由三部分组成：store，reducer，action。 
### store
redux存储数据的地方

要创建一个store，需要用到createStore函数：

```
import { createStore } from 'redux';
const store = createStore(reducer);
```
看一下源码：
```
export const ActionTypes = {
  INIT: '@@redux/INIT'
}

export default function createStore(reducer, preloadedState, enhancer) {
  // 如果 preloadedState类型是function，enhancer类型是undefined，那认为用
  // 户没有传入preloadedState，就将preloadedState的值传给  
  // enhancer，preloadedState值设置为undefined
  if (typeof preloadedState === 'function' && typeof enhancer === 'undefined') {
    enhancer = preloadedState
    preloadedState = undefined
  }
  // enhancer类型必须是一个function
  if (typeof enhancer !== 'undefined') {
    if (typeof enhancer !== 'function') {
      throw new Error('Expected the enhancer to be a function.')
    }
    // 返回使用enhancer增强后的store
    return enhancer(createStore)(reducer, preloadedState)
  }
  // reducer必须是一个function
  if (typeof reducer !== 'function') {
    throw new Error('Expected the reducer to be a function.')
  }

  let currentReducer = reducer
  let currentState = preloadedState
  let currentListeners = []
  let nextListeners = currentListeners
  let isDispatching = false
//...
  dispatch({ type: ActionTypes.INIT })

  return {
    dispatch,
    subscribe,
    getState,
    replaceReducer,
    [$$observable]: observable
  }
```
在初始化时，createStore会主动触发一次dispach，它的action.type是系统内置的INIT，所以在reducer中不会匹配到任何开发者自定义的action.type

createStore会返回一些操作API，包括：

- getState：获取当前的state值
- dispatch：触发reducer并执行listeners中的每一个方法
- subscribe：将方法注册到listeners中，通过dispatch来触发
- replaceReducer:更新当前 store 里的reducer，一般只会在开发模式中调用该方法。
#### getState

```
/**
   * 读取 store 管理的状态树
   *
   * @returns {any} 返回应用中当前的状态树
   */
  function getState() {
    return currentState
  }
```

#### dispatch

```
/**
   * 发送一个 action，这是唯一一种触发状态改变的方法。
   * 每次发送 action，用于创建 store 的 `reducer` 都会被调用一次。调用时传入
   * 的参数是当前的状态以及被发送的 action。的返回值将被当作下一次的状
   * 态，并且监听器将会被通知。
   *
   * 基础实现只支持简单对象的 actions。如果你希望可以发送 
   * Promise，Observable，thunk火气其他形式的 action，你需要用相应的中间
   * 件把 store创建函数封装起来 。例如，你可以参阅 `redux-thunk`包的文档。
   * 不过这些中间件还是通过 dispatch 方法发送简单对象形式的 action。
   * 
   * @param {Object} action，一个标识改变了什么的对象。这是一个很好的点子
   * 保证 actions 可被序列化，这样你就可以记录并且回放用户的操作，或者使用
   * 可以穿梭时间的插件 `redux-devtools`。一个 action 必须有一个值不为
   * `undefined`的type属性，推荐使用字符串常量作为 action types。
   *
   * @returns {Object} 为了方便起见，返回传入的 action 对象。
   *
   * 要注意的是，如果你使用一个自定义的中间件，可能会把`dispatch()`的返回
   * 值封装成其他内容(比如，一个可以await的Promise)。
   */
  function dispatch(action) {
    // 如果 action不是一个简单对象，抛出异常
    if (!isPlainObject(action)) {
      throw new Error(
        'Actions must be plain objects. ' +
        'Use custom middleware for async actions.'
      )
    }

    if (typeof action.type === 'undefined') {
      throw new Error(
        'Actions may not have an undefined "type" property. ' +
        'Have you misspelled a constant?'
      )
    }
    
    // reducer内部不允许再次调用dispatch，否则抛出异常
    if (isDispatching) {
      throw new Error('Reducers may not dispatch actions.')
    }

    try {
      isDispatching = true
      currentState = currentReducer(currentState, action)
    } finally {
      isDispatching = false
    }

    const listeners = currentListeners = nextListeners
    for (let i = 0; i < listeners.length; i++) {
      const listener = listeners[i]
      listener()
    }

    return action
  }

```

#### subscribe

```
/**
   * 添加一个改变监听器。它将在一个action被分发的时候触发，并且状态数的某
   * 些部分可能已经发生了变化。那么你可以调用 getState 来读取回调中的当前
   * 状态树。
   *    
   * 你可以从一个改变的监听中调用 dispatch()，注意事项：
   *
   * 1.在每一次调用 dispatch() 之前监听器数组都会被复制一份。如果你在监听函
   * 数中订阅或者取消订阅，这个不会影响当前正在进行的 dispatch()。而下次
  * dispatch()是否是嵌套调用，都会使用最新的修改后的监听列表。
   * 2.监听器不希望看到哦啊所有状态的改变，如状态可能在监听器被调用前可能
   * 在嵌套 dispatch() 可能更新过多次。但是，在某次dispatch
   * 触发之前已经注册的监听函数都可以读取到这次diapatch之后store的最新状
   * 态。
   *
   * @param {Function} listener 在每次 dispatch 之后会执行的回调函数。
   * @returns {Function} 返回一个用于取消这次订阅的函数。
   */
  function subscribe(listener) {
    if (typeof listener !== 'function') {
      throw new Error('Expected listener to be a function.')
    }

    let isSubscribed = true

    ensureCanMutateNextListeners()
    nextListeners.push(listener)

    return function unsubscribe() {
      if (!isSubscribed) {
        return
      }

      isSubscribed = false

      ensureCanMutateNextListeners()
      const index = nextListeners.indexOf(listener)
      nextListeners.splice(index, 1)
    }
  }

```

#### replaceReducer

```
/**
   * 替换 store 当前使用的 reducer 函数。
   *
   * 如果你的程序代码实现了代码拆分，并且你希望动态加载某些 reducers。或
   * 者你为 redux 实现一个热加载的时候，你也会用到它。
   *
   * @param {Function} nextReducer 替换后的reducer
   * @returns {void}
   */
  function replaceReducer(nextReducer) {
    if (typeof nextReducer !== 'function') {
      throw new Error('Expected the nextReducer to be a function.')
    }

    currentReducer = nextReducer
    dispatch({ type: ActionTypes.INIT })
  }

```

### action

```
const ADD_TODO = 'ADD_TODO'
{
  type: ADD_TODO,
  text: 'Build my first Redux app'
}
```
action 内必须使用一个字符串类型的 type 字段来表示将要执行的动作。
### reducer

Reducers 指定了应用状态的变化如何响应 actions 并发送到 store 的

reducer 就是一个纯函数，接收旧的 state 和 action，返回新的 state：

```
(previousState, action) => newState
```
#### 使用

```
function todoApp(state = initialState, action) {
  switch (action.type) {
    case SET_VISIBILITY_FILTER:
      return Object.assign({}, state, {
        visibilityFilter: action.filter
      })
    default:
      return state
  }
}
```
#### 注意:

不要修改 state。 使用 Object.assign() 新建了一个副本。不能这样使用 Object.assign(state, { visibilityFilter: action.filter })，因为它会改变第一个参数的值。你必须把第一个参数设置为空对象。你也可以开启对ES7提案对象展开运算符的支持, 从而使用 { ...state, ...newState } 达到相同的目的。

在 default 情况下返回旧的 state。遇到未知的 action 时，一定要返回旧的 state。
#### 不要在reducer执行的操作
修改传入参数；
执行有副作用的操作，如 API 请求和路由跳转；
调用非纯函数，如 Date.now() 或 Math.random()。

## 整个流程代码

```
//index.js
import { createStore } from 'redux'
import todoApp from './reducers'

let store = createStore(todoApp)
//child component
import {
  addTodo,
  toggleTodo,
  setVisibilityFilter,
  VisibilityFilters
} from './actions'

// 打印初始状态
console.log(store.getState())

// 每次 state 更新时，打印日志
// 注意 subscribe() 返回一个函数用来注销监听器
const unsubscribe = store.subscribe(() =>
  console.log(store.getState())
)

// 发起一系列 action
store.dispatch(addTodo('Learn about actions'))
store.dispatch(addTodo('Learn about reducers'))
store.dispatch(addTodo('Learn about store'))
store.dispatch(toggleTodo(0))
store.dispatch(toggleTodo(1))
store.dispatch(setVisibilityFilter(VisibilityFilters.SHOW_COMPLETED))

// 停止监听 state 更新
unsubscribe();
```
## React-redux
react中可以通过react-redux来连接redux, 这样一来我们就不需要一层层的传递store对象了

`<Provider store>` 使组件层级中的 connect() 方法都能够获得 Redux store。

React-Redux提供一个connect方法能够让你把组件和store连接起来。
```
ReactDOM.render(
  <Provider store={store}>
    <MyRootComponent />
  </Provider>,
  rootEl
)


connect([mapStateToProps], [mapDispatchToProps], [mergeProps], [options])

```



### mapStateToProps
> mapStateToProps应该声明为一个方法：
> 
> function mapStateToProps(state, ownProps?)
> 他接收的第一个参数是state，可选的第二个参数时ownProps，然后返回一个被连接组件所需要的数据的纯对象。
> 
> 这个方法应作为第一个参数传递给connect，并且会在每次Redux store state改变时被调用。如果你不希望订阅store，那么请传递null或者undefined替代mapStateToProps作为connect的第一个参数。
> 
> 无论mapStateToProps是使用function关键字声明的(function mapState(state) { } ) 还是以一个箭头函数(const mapState = (state) => { } ) 的形式定义的——它都能够以同样的方式生效。
> 
> 参数：
> - state
> - ownProps（可选）

mapStateToProps就是将redux里面的state转换为react里的props并传入组件


mapStateToProps的第一个参数是整个Redux store state对象

### mapDispatchToProps
就是把各种dispatch也变成了props让你可以直接使用

```
const mapDispatchToProps = (dispatch) => {
  return {
    onClick: () => {
      dispatch({
        type: 'act',
　　　　  value : 100
      });
    }
  };
}
```



### 完整栗子
#### action
```
export const Increment='increment'
export const Decrement='decrement'

export const increment=(counterCaption)=>({
    type:Increment,
    counterCaption
  }
)
export const decrement=(counterCaption)=>({
    type:Decrement,
    counterCaption
})
```

#### Reducer


```
import {Increment,Decrement} from '../Action'
export default(state,action)=>{
    const {counterCaption}=action
    switch (action.type){
        case Increment:
            return {...state,[counterCaption]:state[counterCaption]+1}
        case Decrement:
            return {...state,[counterCaption]:state[counterCaption]-1}
        default:
            return state
    }
}
```

#### store


```
import {createStore} from 'redux'
import reducer from '../Reducer' 
const initValue={
    'First':0,
    'Second':10,
    'Third':20
}
const store=createStore(reducer,initValue)
export default store
```


#### counter组件
```
import React, { Component } from 'react'
import {increment,decrement} from '../Redux/Action'
import {connect} from 'react-redux';
const buttonStyle = {
    margin: "20px"
}

function Counter({caption, Increment, Decrement, value}){
    return (
        <div>
            <button style={buttonStyle} onClick={Increment}>+</button>
            <button style={buttonStyle} onClick={Decrement}>-</button>
            <span>{caption} count :{value}</span>
        </div>
    )
}
function mapState(state,ownProps){
    return{
        value:state[ownProps.caption]
    }
}
function mapDispatch(dispatch,ownProps){
    return {
        Increment:()=>{
            dispatch(increment(ownProps.caption))
        },
        Decrement:()=>{
            dispatch(decrement(ownProps.caption))
        }
    
    }
}

export default connect(mapState,mapDispatch)(Counter)
```
#### controlpanel组件

```
import React, { Component } from 'react'
import Counter from './Counter.jsx'
import Summary from './Summary.jsx'
const style = {
    margin: "20px"
}

class ControlPanel extends Component {
    render() {
        return (
            <div style={style}>
                <Counter caption="First" />
                <Counter caption="Second"/>
                <Counter caption="Third" />
                <hr/>
                <Summary/>
            </div>
        )
    }
}
export default ControlPanel
```


#### app.js
```
import React, {Component, PropTypes} from 'react';
import ReactDOM, {render} from 'react-dom';
import store from './Redux/Store/Store.jsx'
import {Provider} from 'react-redux';
import ControlPanel from './Component/ControlPanel.jsx'
import './style/common.less'
render(
    <Provider store={store}>
        <ControlPanel />
    </Provider>,
    document.body.appendChild(document.createElement('div'))
);
```
## 参考文章
https://segmentfault.com/a/1190000017064759?utm_source=tag-newest#articleHeader3

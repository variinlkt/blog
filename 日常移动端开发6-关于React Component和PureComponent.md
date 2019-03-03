# React Component和PureComponent
## PureComponent是什么
简而言之，PureComponent是用于react 性能优化的

当我们在调用setState改变state以后，在调用render渲染到页面之前，react做了以下几件事：
- 若有shouldComponentUpdate，则执行它
- 若没有这个方法会判断是不是PureComponent，若是，进行浅比较
- 浅比较时，如果返回true则表示数据相等，否则表示不等

```
if (this._compositeType === CompositeTypes.PureClass) {
  shouldUpdate = !shallowEqual(prevProps, nextProps) || ! shallowEqual(inst.state, nextState);
}
```

### 浅比较
浅比较(shallowEqual)，即react源码中的一个函数

```
// 用原型链的方法
const hasOwn = Object.prototype.hasOwnProperty

// 这个函数实际上是Object.is()的polyfill
function is(x, y) {
  if (x === y) {
    return x !== 0 || y !== 0 || 1 / x === 1 / y
  } else {
    return x !== x && y !== y
  }
}

export default function shallowEqual(objA, objB) {
  // 首先对基本数据类型的比较
  if (is(objA, objB)) return true
  // 由于Obejct.is()可以对基本数据类型做一个精确的比较， 所以如果不等
  // 只有一种情况是误判的，那就是object,所以在判断两个对象都不是object
  // 之后，就可以返回false了
  if (typeof objA !== 'object' || objA === null ||
      typeof objB !== 'object' || objB === null) {
    return false
  }

  // 过滤掉基本数据类型之后，就是对对象的比较了
  // 首先拿出key值，对key的长度进行对比

  const keysA = Object.keys(objA)
  const keysB = Object.keys(objB)

  // 长度不等直接返回false
  if (keysA.length !== keysB.length) return false
  // key相等的情况下，在去循环比较
  for (let i = 0; i < keysA.length; i++) {
  // key值相等的时候
  // 借用原型链上真正的 hasOwnProperty 方法，判断ObjB里面是否有A的key的key值
  // 属性的顺序不影响结果也就是{name:'daisy', age:'24'} 跟{age:'24'，name:'daisy' }是一样的
  // 最后，对对象的value进行一个基本数据类型的比较，返回结果
    if (!hasOwn.call(objB, keysA[i]) ||
        !is(objA[keysA[i]], objB[keysA[i]])) {
      return false
    }
  }

  return true
}
```
总结：shallowEqual会比较 Object.keys(state | props) 的长度是否一致，每一个 key 是否两者都有，并且是否是一个引用，也就是只比较了第一层的值
#### 浅比较为什么没办法对嵌套的对象比较？

由上面的分析可以看到，当对比的类型为Object的时候并且key的长度相等的时候，浅比较也仅仅是用`Object.is()`对Object的value做了一个基本数据类型的比较，所以如果key里面是对象的话，有可能出现比较不符合预期的情况，所以浅比较是不适用于嵌套类型的比较的。

## Component和PureComponent区别
- 当组件更新时，PureComponent多了一层浅比较，只有在数据不相等时（基础数据类型不相等；引用类型不属于同一个引用并且第一层的kv值不相等）才会调用render函数渲染视图。如果组件的 props 和 state 都没发生改变， render 方法就不会触发，省去 Virtual DOM 的生成和比对过程，达到提升性能的目的
- Component则没有浅比较的过程
## 使用PureComponent需要注意的地方
- 继承PureComponent时，不能再重写shouldComponentUpdate
- 易变数据不能使用一个引用，对于数组可以使用slice或concat来获取一个新引用
## 兼容写法
```
import React { PureComponent, Component } from 'react';

class Foo extends (PureComponent || Component) {
  //...
}
```

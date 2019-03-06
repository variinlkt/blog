# setState机制
## 源码
```
ReactComponent.prototype.setState = function(partialState, callback) {
  invariant(
    typeof partialState === 'object' ||
      typeof partialState === 'function' ||
      partialState == null,
    'setState(...): takes an object of state variables to update or a ' +
      'function which returns an object of state variables.',
  );
  this.updater.enqueueSetState(this, partialState);
  if (callback) {
    this.updater.enqueueCallback(this, callback, 'setState');
  }
};
 enqueueSetState: function(publicInstance, partialState) {
    if (__DEV__) {
      ReactInstrumentation.debugTool.onSetState();
      warning(
        partialState != null,
        'setState(...): You passed an undefined or null state object; ' +
          'instead, use forceUpdate().',
      );
    }

    var internalInstance = getInternalInstanceReadyForUpdate(
      publicInstance,
      'setState',
    );

    if (!internalInstance) {
      return;
    }

    var queue =
      internalInstance._pendingStateQueue ||
      (internalInstance._pendingStateQueue = []);
    queue.push(partialState);

    enqueueUpdate(internalInstance);
  }
// 通过enqueueUpdate执行state更新
function enqueueUpdate(component) {
  ensureInjected();
  // batchingStrategy是批量更新策略，isBatchingUpdates表示是否处于批量更新过程
  // 最开始默认值为false
  if (!batchingStrategy.isBatchingUpdates) {
    batchingStrategy.batchedUpdates(enqueueUpdate, component);
    return;
  }
  dirtyComponents.push(component);

  if (component._updateBatchNumber == null) {
    component._updateBatchNumber = updateBatchNumber + 1;
  }
}
// 对_pendingElement, _pendingStateQueue, _pendingForceUpdate进行判断，
// _pendingStateQueue由于会对state进行修改，所以不为空，
// 然后会调用updateComponent方法
performUpdateIfNecessary: function(transaction) {
    if (this._pendingElement != null) {
      ReactReconciler.receiveComponent(
        this,
        this._pendingElement,
        transaction,
        this._context,
      );
    } else if (this._pendingStateQueue !== null || this._pendingForceUpdate) {
      this.updateComponent(
        transaction,
        this._currentElement,
        this._currentElement,
        this._context,
        this._context,
      );
    } else {
      this._updateBatchNumber = null;
    }
  },
```
上面这段代码的意思就是如果是处于批量更新模式，也就是isBatchingUpdates为true时，不进行state的更新操作，而是将需要更新的component添加到dirtyComponents数组中。
如果不处于批量更新模式，则对所有队列中的更新执行batchedUpdates方法。
当我们调用setState时，最终会通过enqueueUpdate执行state更新，就像上面那样有两种更新的模式，一种是批量更新模式，将组建保存在dirtyComponents;另一种非批量模式，将会遍历dirtyComponents，对每一个dirtyComponents调用updateComponent方法。就像这张图：

![](https://github.com/variinlkt/blog/blob/master/imgs/4116027-87196475cca24c19.jpeg)

在非批量模式下，会立即应用新的state；而在批量模式下，需要更新state的组件会被push 到dirtyComponents，再执行更新。
## setState的基本过程

setState的调用会引起React的更新生命周期的4个函数执行。

- shouldComponentUpdate
- componentWillUpdate
- render
- componentDidUpdate

当shouldComponentUpdate执行时，返回true，进行下一步，this.state没有被更新
返回false，停止，更新this.state

当componentWillUpdate被调用时，this.state也没有被更新

直到render被调用时候，this.state才被更新。

总之，**直到下一次render函数调用(或者下一次shouldComponentUpdate返回false时)才能得到更新后的this.state**
## 如何在setState 后直接获取修改后的值

- 传入对应的参数，不通过 this.state 获取


- 使用回调函数
setState 方法接收一个 function 作为回调函数。这个回调函数会在 setState 完成以后直接调用，这样就可以获取最新的 state 。

- 使用setTimeout
在 setState 使用 setTimeout 来让 setState 先完成以后再执行里面内容。
## setState的使用和特点

```
setState(object,[callback]) //对象式，object为nextState
setState(function,[callback]) //函数式，function为(prevState,props) => stateChange
```

**[callback]则为state更新之后的回调，此时state已经完成更新，可以取到更新后的state**
[callback]是在setState之后，更准确来说是当正式执行batchUpdate队列的state更新完成后就会执行，不是在re-rendered之后

使用两种形式的setState，state的更新都是异步的，但是
### 多次连续使用函数式的setState，上一次函数执行后的state传给下一个函数，因此每次执行setState后能读取到更新后的state值
React本身会进行一个递归传递调用，将上一次函数执行后的state传给下一个函数，因此每次执行
setState后能读取到更新后的state值。

```
this.setState(prevState => {
    return {index: prevState.index + 1};
});
this.setState(prevState => {
    return {index: prevState.index + 1};
});
```
#### 原因
在setState的第一个参数中传入function，该function会被压入调用栈中，在state真正改变后，按顺序回调栈里面的function。该function的第一个参数为上一次更新后的state。这样就能确保你下一次的操作拿到的是你预期的值
### **如果对象式和函数式的setState混合使用，则对象式的会覆盖前面无论函数式还是对象式的任何setState，**
但是不会影响后面的setState。


```
function increment(state,props){
    return {count: state.count + 1};
}

function incrementMultiple(){
    this.setState(increment);
    this.setState(increment);
    this.setState({count: this.state.count + 1});
    this.setState(increment);
}
```

上面三个函数式的setState中间插入一个对象式的setState，则最后的结果是2，而不是4，

因为对象式的setState将前面的任何形式的setState覆盖了，但是后面的setState依然起作用

### 函数式的setState的使用场景
this.props 和 this.state 可能是异步更新的，你不应该依靠它们的值来计算下一个状态。

例如，此代码可能无法更新计数器：


```
// Wrong
this.setState({
  counter: this.state.counter + this.props.increment,
});
```

要修复它，请使用第二种形式的 setState() 来接受一个函数而不是一个对象。 该函数将接收先前的状态作为第一个参数，将此次更新被应用时的props做为第二个参数：


```
// Correct
this.setState((prevState, props) => ({
  counter: prevState.counter + props.increment
}));
```


### 在mount流程中调用setState，不会进入update流程，队列在mount时合并修改并render

不多说



### setState会合并修改
当遇到多个setState调用时候，会提取单次传递setState的对象，把他们合并在一起形成一个新的
单一对象，并用这个单一的对象去做setState的事情，就像Object.assign的对象合并，后一个
key值会覆盖前面的key值

```
const a = {name : 'kong', age : '17'}
const b = {name : 'fang', sex : 'men'}
Object.assign({}, a, b);
//{name : 'fang', age : '17', sex : 'men'}
```

name被后面的覆盖了，但是age和sex都起作用了

下面的代码，第一次和第二次的setState合并了，最后只加了1
```
class Example extends React.Component {
  constructor() {
    super();
    this.state = {
      val: 0
    };
  }
  
  componentDidMount() {
    this.setState({val: this.state.val + 1});
    console.log(this.state.val);    // 第 1 次 log

    this.setState({val: this.state.val + 1});
    console.log(this.state.val);    // 第 2 次 log

    setTimeout(() => {
      this.setState({val: this.state.val + 1});
      console.log(this.state.val);  // 第 3 次 log

      this.setState({val: this.state.val + 1});
      console.log(this.state.val);  // 第 4 次 log
    }, 0);
  }
  render() {
    return null;
  }
};

//4次log的值 分别为 0 0 2 3
```
总结起来就是这样：

- this.setState首先会把state推入pendingState队列中
然后将组件标记为dirty
- React中有事务的概念，最常见的就是更新事务，如果不在事务中，则会开启一次新的更新事务，更新事务执行的操作就是把组件标记为dirty。
- 判断是否处于batch update
- 是的话，保存组建于dirtyComponent中，在事务结束的时候才会通过 ReactUpdates.flushBatchedUpdates 方法将所有的临时 state merge 并计算出最新的 props 及 state，然后将其批量执行，最后再关闭结束事务。
- 不是的话，直接开启一次新的更新事务，在标记为dirty之后，直接开始更新组件。因此当setState执行完毕后，组件就更新完毕了，所以会造成定时器同步更新的情况。

### 不要直接修改this.state
#### 为什么直接修改this.state无效

要知道setState本质是通过一个队列机制实现state更新的。 执行setState时，会将需要更新的state合并后放入状态队列，而不会立刻更新state，队列机制可以批量更新state。
如果不通过setState而直接修改this.state，那么这个state不会放入状态队列中，下次调用setState时对状态队列进行合并时，会忽略之前直接被修改的state，这样我们就无法合并了，而且实际也没有把你想要的state更新上去。



## setState要注意的地方

**不要在shouldComponentUpdate和componentWillUpdate中调用setState**
```
{   // 会检测组件中的state和props是否发生变化，有变化才会进行更新; 
    // 如果shouldUpdateComponent函数中返回false则不会执行组件的更新
    updateComponent: function (transaction,
                               prevParentElement,
                               nextParentElement,
                               prevUnmaskedContext,
                               nextUnmaskedContext,) {
        var inst = this._instance;
        var nextState = this._processPendingState(nextProps, nextContext);
        var shouldUpdate = true;

        if (!this._pendingForceUpdate) {
            if (inst.shouldComponentUpdate) {
                if (__DEV__) {
                    shouldUpdate = measureLifeCyclePerf(
                            () => inst.shouldComponentUpdate(nextProps, nextState, nextContext),
                            this._debugID,
                            'shouldComponentUpdate',
                    );
                } else {
                    shouldUpdate = inst.shouldComponentUpdate(
                            nextProps,
                            nextState,
                            nextContext,
                    );
                }
            } else {
                if (this._compositeType === CompositeTypes.PureClass) {
                    shouldUpdate =
                            !shallowEqual(prevProps, nextProps) ||
                            !shallowEqual(inst.state, nextState);
                }
            }
        }
        
    },
// 该方法会合并需要更新的state，然后加入到更新队列中
    _processPendingState: function (props, context) {
        var inst = this._instance;
        var queue = this._pendingStateQueue;  
        var replace = this._pendingReplaceState;
        this._pendingReplaceState = false; 
        this._pendingStateQueue = null;

        if (!queue) {
            return inst.state;
        }

        if (replace && queue.length === 1) {
            return queue[0];
        }

        var nextState = Object.assign({}, replace ? queue[0] : inst.state);
        for (var i = replace ? 1 : 0; i < queue.length; i++) {
            var partial = queue[i];
            Object.assign(
                    nextState,
                    typeof partial === 'function'
                            ? partial.call(inst, nextState, props, context)
                            : partial,
            );
        }

        return nextState;
    }
};


```

当调用setState时，实际上会执行enqueueSetState方法，并对partialState以及_pendingStateQueue更新队列进行合并，最终通过enqueueUpdate执行state更新。

而performUpdateIfNecessary方法获取_pendingElement、_pendingStateQueue、_pendingForceUpdate，并调用reciveComponent和updateComponent方法进行组件更新。

如果在shouldComponentUpdate或者componentWillUpdate方法中调用setState，此时this._pendingStateQueue != null，则perfromUpdateIfNecessary方法就会调用updateComponent方法进行组件更新，但updateComponent方法又会调用shouldComponentUpdate和componentWillUpdate方法，造成循环调用。


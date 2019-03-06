# react事件系统-1
好久之前写过的，这里再总结一下8
## react事件流
react有一个合成事件（SyntheticEvent）的概念

合成事件指的是在jsx中直接绑定的事件，如

```
<a ref="aaa" onClick={(e)=>this.handleClick(e)}>更新</a>
```

原生事件指的是通过js原生代码绑定的事件，如

```
this.refs.update.addEventListener('click',e=>{
     console.log('update');
 });
```

如果想在react里面访问当原生的事件对象，可以通过引用**nativeEvent**获得。

**通过jsx绑定的事件，react会将将事件委托给父元素处理，即会将事件绑定到document上（但是并不是直接将事件代理在document上，而是绑定了一个dispatchInteractiveEvent 函数），在冒泡阶段处理事件**

react这样做的原因是：**在挂载或者卸载组件时，只需要在通过的在统一的事件监听位置增加或者删除对象，可以提高效率。**

### 事件流顺序：

```
class Index extends React.Component{
    clickC(e){
        console.log('child')
        e.stopPropagation()
    }
    clickP(e){
        console.log('parent')
    }
    componentDidMount(){
        //原生事件绑定在#child上
        //原生事件绑定在document上
    }
    render(){
        return (
        <div>
            <div id='parent' onClick={this.clickP.bind(this)}></div>
            <div id='child' onClick={this.clickC.bind(this)}></div>
        </div>
        )
    }
}
```

执行顺序：

1、直接绑定在#child上的事件

2、输出child

3、直接绑定在document的事件

因此，在 React 中，想要阻止“事件冒泡”（React 只是模拟事件冒泡，并非真正的 DOM 事件冒泡），只需要在回调函数中调用 e.stopPropagation（这时候的 e.stopPropagation非原生事件对象的 stopPropagation）

1. 事件流首先进入到 #child ，然后触发直接绑定在 #child 上的事件；
2. 事件流沿着 DOM 结构向上冒泡到 document，触发 React 绑定的 dispatchInteractiveEvent 函数，从而调用了 #child 子元素上绑定的 clickChild 方法。
3. 在 clickChild 方法的最后，我调用了 e.stopPropagation，成功地阻止了 React 模拟的事件冒泡，因此，成功地没有触发 #parent 上的事件。
4. 最后还是触发了 document 上直接绑定的事件。如果不要触发任何其他元素的事件，包括 document，应该调用e.nativeEvent.stopImmediatePropagation(假设事件流已经被某个元素捕获（或者冒泡到某个元素），那么便会触发此元素上绑定的事件。如果绑定的事件不止一个，则依次触发。假如想中断这种依次触发，可以调用 e.stopImmediatePropagation。)
￼
![](https://dfiles.tita.com/Portal/110006/356040fa3dc8469c8258d8f80f2814bc.png)
#### 总结：如何在react中阻止冒泡事件
A、阻止合成事件间的冒泡，用`e.stopPropagation()`;
 
B、阻止合成事件与最外层document上的事件间的冒泡，用`e.nativeEvent.stopImmediatePropagation()`;
 
C、阻止合成事件与除最外层document上的原生事件上的冒泡，通过判断`e.target`来避免
 

```
class Counter extends Component{
//....
componentDidMount() {
document.body.addEventListener('click',e=>{
// 通过e.target判断阻止冒泡
    if(e.target&&e.target.matches('a')){
        return;
     }
    console.log('body');
 })
 }
}
```

然后我去看了下react源码，大概的实现原理是：

**react将props解析成react event ，利用了registrationNameDependencies——一个存储了 React事件名与浏览器原生事件名对应的一个 Map，可以通过这个 map拿到相应的浏览器原生事件名，再注册原生事件**

## 事件系统
1、使用事件委托技术进行事件代理，React 组件上声明的事件最终都转化为 DOM 原生事件，绑定到了 document 这个 DOM 节点上。从而减少了内存开销。

2、自身实现了一套事件冒泡机制，以队列形式，从触发事件的组件向父组件回溯，调用在 JSX 中绑定的 callback。因此我们也没法用 event.stopPropagation() 来停止事件传播，应该使用 React 定义的 event.preventDefault()。

3、React 有一套自己的合成事件 SyntheticEvent，而不是单纯的使用 DOM 原生事件，但二者可以平滑转化。

4、React 使用事件池来管理合成事件对象的创建和销毁，这样减少了垃圾的生成和新对象内存的分配，大大提高了性能。

### 事件注册
事件注册即在 document 节点，将 React 事件转化为 DOM 原生事件，并注册回调。
#### ReactDomComponent.js

![](https://dfiles.tita.com/Portal/110006/4cc68279ce524ede95e144943eecaf53.png)

#### ReactBrowserEventEmitter.js
![](https://dfiles.tita.com/Portal/110006/8763cce1a52f41008e0fde22f265d9a2.png)

#### ReactEventListener.js
![](https://dfiles.tita.com/Portal/110006/563738df83414bb3ad0f2b891a611ac3.png)
#### EventListener.js
![](https://dfiles.tita.com/Portal/110006/14032cc3fa0a47909534d349c3cc974a.png)
#### 事件注册总结
**将注册的react事件转化为dom原生事件，并在document节点绑定事件，事件触发时调用dispatchEvent方法**
### 事件存储

事件注册之后，还需要将事件绑定的回调函数存储下来。这样，在触发事件后才能去寻找相应回调来触发。
#### ReactDomComponent.js

使用 putListener 方法来进行事件回调存储。

```
var listenerBank = {};
var getDictionaryKey = function (inst) {
//inst为组建的实例化对象
//_rootNodeID为组件的唯一标识
  return '.' + inst._rootNodeID;
}
var EventPluginHub = {
//inst为组建的实例化对象
//registrationName为事件名称
//listner为我们写的回调函数，也就是列子中的this.autoFocus
  putListener: function (inst, registrationName, listener) {
    ...
    var key = getDictionaryKey(inst);
    var bankForRegistrationName = listenerBank[registrationName] || (listenerBank[registrationName] = {});
    bankForRegistrationName[key] = listener;
    ...
  }
}
```

#### 事件存储总结
将事件的回调函数放入listenerbank中

### 事件触发

当我们点击一个按钮时，click事件将会最终冒泡至document，并触发我们监听在document上的handler 

#### dispatchEventEventListener.js
![](https://dfiles.tita.com/Portal/110006/2b4c3cc1d1894be39ef093d856ff1299.png)

这里的getPooled函数是判断事件池里面有没有可用的事件，有就复用，没有就新建，**后面会讲到**

![](https://dfiles.tita.com/Portal/110006/58e57949e0bd45a4b399a32b19b93670.png)

handleTopLevelImpl调用了ReactEventListener._handleTopLevel，**它可以获取合成事件，并且去执行它。**

#### 执行过程：

#### ReactEventEmitterMixin.js

![](https://dfiles.tita.com/Portal/110006/ffbe22bab3554882a2c3a61fc38fb99d.png)

handleTopLevel会调用runExtractedEventsInBatch()，它做了两件事：

1、根据原生事件合成合成事件，并且在vDOM上模拟捕获冒泡，收集所有需要执行的事件回调构成回调数组。

2、遍历回调数组，触发回调函数。

其中extractEvents即TapEventPlugin.js里面createTapEventPlugin()返回的对象的方法



#### EventPluginHub.js
![](https://dfiles.tita.com/Portal/110006/38f8ddb544274ff39dc0091e4653d7db.png)

react有合成事件synthetic，合成事件是 pooled（循环使用的），这意味着合成事件对象会被重复使用，所有的属性在被调用以后会被值为null，该机制用于性能优化，因此你不可以异步访问事件。除非调用 event.persist()，该方法不会把事件放入事件池中，保持event对象不被重置允许代码的引用到。

![](https://dfiles.tita.com/Portal/110006/24ba1388deb64fc296536b385e8a0543.png)

forEachAccumulated这个函数，就是对数组processingEventQueue的每一个合成事件都使用executeDispatchesAndReleaseTopLevel来dispatch 事件。


#### dispatch全过程：

#### EventPluginUtils.js
![](https://dfiles.tita.com/Portal/110006/9eee4ef66e4e4b7888222e85e406ca49.png)

**dispatch 合成事件分为两个步骤：**

- 通过_dispatchListeners里得到所有绑定的回调函数，
- 再通过_dispatchInstances得到所有绑定了回调函数的虚拟dom元素
循环执行_dispatchListeners里所有的回调函数，如果回调函数里使用了stopPropagation会使得数组后面的回调函数不能执行，这样就做到了阻止事件冒泡。

#### EventPluginHub.js
![](https://dfiles.tita.com/Portal/110006/fce2c0961d9c4d51b182447dbda9c655.png)

将事件对应的dom元素绑定到了currentTarget上， 这样我们通过e.currentTarget就可以找到绑定事件的原生dom元素。
#### ReactErrorUtils.js
![](https://dfiles.tita.com/Portal/110006/fb968254dd6741129f5076f5ae5c3d92.png)

调用了faked元素的dispatchEvent方法来触发事件，并且触发完毕之后立即移除监听事件。


### 总结：整个click事件被分发的过程就是：

1、用EventPluginHub生成合成事件，同一事件类型只会生成一个合成事件，里面的_dispatchListeners里储存了同一事件类型的所有回调函数

2、按顺序去执行它

## 总总结：
### 流程图
![](https://github.com/variinlkt/blog/blob/master/imgs/2582771838-59cdeeb8f2084_articlex.jpeg)
### listenerBank、registrationNameDependencies和event._dispatchListeners的区别？
listenerbank是一个对象，存储了单个react事件的回调函数
event._dispatchListeners是一个数组，存储了同一类型事件 的所有回调函数，在dispatch时按顺序执行他们
registrationNameDependencies是一个存储了 React事件名与浏览器原生事件名对应的一个 Map

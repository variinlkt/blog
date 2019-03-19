# react-native使用小结
## Animated API动画
### 使用

```
import { Animated } from 'react-native'
Animated.timing(
  // timing方法使动画值随时间变化
  this.state.fadeAnim, // 要变化的动画值
  {
    toValue: 1, // 最终的动画值
  },
).start(); // 开始执行动画

//inside constructor
this.state = {
    fadeAnim: new Animated.Value(0)
}
```
- start()启动动画
- stop()停止动画
#### 循环播放
loop()

```
static loop(animation, config?)
```

无限循环一个指定的动画，从头到尾周而复始。

config参数：
- iterations: Number of times the animation should loop. Default -1 (infinite).

```
Animated.loop(animation).start()
```

### 动画类型
- Animated.decay()以指定的初始速度开始变化，然后变化速度越来越慢直至停下。
- Animated.spring()提供了一个简单的弹簧物理模型.
- Animated.timing()使用easing 函数让数值随时间动起来。

```
static timing(value, config)
```

Config 参数有以下这些属性：

- duration: 动画的持续时间（毫秒）。默认值为 500.
- easing: 缓动函数。 默认为Easing.inOut(Easing.ease)。
- delay: 开始动画前的延迟时间（毫秒）。默认为 0.
- isInteraction: 指定本动画是否在InteractionManager的队列中注册以影响其任务调度。默认值为 true。
- useNativeDriver: 启用原生动画驱动。默认不启用(false)。

### 实现动画的组件
- Animated.Image
- Animated.ScrollView
- Animated.Text
- Animated.View

#### 封装未实现动画的组件
createAnimatedComponent()方法正是用来处理组件，使其可以用于动画。

```
class Sub extends PureComponent {

    render() {
        const {width, height} = this.props
        return (
            <View style={{width,height,backgroundColor: 'blue'}}>

            </View>
        );
    }
}

const ABC = Animated.createAnimatedComponent(Sub)


constructor(props) {
        super(props);
        this.state = {
            width: new Animated.Value(0),
            height: new Animated.Value(0)
        }
    }


    componentDidMount(){
        Animated.parallel([
            Animated.timing(this.state.width,{
                toValue: 200,
                duration: 5000
            }),
            Animated.timing(this.state.height,{
                toValue: 200,
                duration: 5000
            })
        ]).start()
    }

<ABC width={this.state.width} height={this.state.height}/>
```

### 组合动画
- Animated.delay()在给定延迟后开始动画。
- Animated.parallel()同时启动多个动画。
- Animated.sequence()按顺序启动动画，等待每一个动画完成后再开始下一个动画。
- Animated.stagger()按照给定的延时间隔，顺序并行的启动动画。
默认情况下，如果任何一个动画被停止或中断了，组内所有其它的动画也会被停止。Parallel 有一个stopTogether属性，如果设置为false，可以禁用自动停止。


### 插值
interpolate()函数允许输入范围映射到不同的输出范围。默认情况下，它将推断超出给定范围的曲线，但也可以限制输出值。它默认使用线性插值，但也支持缓动功能。
```
<Animated.Image source={require('../assets/img/loading.png')} 
    style={[styles.image, {
        transform: [{
            rotate: degree.interpolate({
                inputRange: [0, 360],
                outputRange: ['0deg', '360deg']
            })
        }]
    }]}>
</Animated.Image>
```
这里处理transform.rotate需要用到interpolate函数
### 处理手势和其他事件
手势，如平移或滚动，以及其他事件可以使用Animated.event()直接映射到动画值。这是通过结构化映射语法完成的，以便可以从复杂的事件对象中提取值。第一层参数是一个数组，你可以在其中指定多个参数映射，这种映射可以是嵌套的对象。

```
static event(argMapping, config?)
```

接受一个映射的数组，对应的解开每个值，然后调用所有对应的输出的setValue方法。例如：

```
onScroll={Animated.event(
   [{
        nativeEvent: {
            contentOffset: {
                x: this._scrollX
            }
        }
   }],
   {listener: (event) => console.log(event)}, // 可选的异步监听函数
 )}
 ...
 onPanResponderMove: Animated.event([
   null,                // 忽略原始事件
   {dx: this._panX}],    // 手势状态参数
    {
        listener: (event, gestureState) => console.log(event, gestureState)
        
    }, // 可选的异步监听函数
 ),
```

Config 参数有以下这些属性：

- listener: 可选的异步监听函数
- useNativeDriver: 启用原生动画驱动。默认不启用(false)。

### 合成动画值
你可以使用加减乘除以及取余等运算来把两个动画值合成为一个新的动画值：

- Animated.add()
- Animated.subtract()
- Animated.divide()
- Animated.modulo()
- Animated.multiply()

### 启用原生动画驱动
Animated的 API 是可序列化的（即可转化为字符串表达以便通信或存储）。通过启用原生驱动，我们在启动动画前就把其所有配置信息都发送到原生端，利用原生代码在 UI 线程执行动画，而不用每一帧都在两端间来回沟通。如此一来，**动画一开始就完全脱离了 JS 线程，因此此时即便 JS 线程被卡住，也不会影响到动画了。**

在动画中启用原生驱动非常简单。只需在开始动画之前，在动画配置中加入一行`useNativeDriver: true`，如下所示：

```
Animated.timing(this.state.animatedValue, {
  toValue: 1,
  duration: 500,
  useNativeDriver: true // <-- 加上这一行
}).start();
```

动画值在不同的驱动方式之间是不能兼容的。**因此如果你在某个动画中启用了原生驱动，那么所有和此动画依赖相同动画值的其他动画也必须启用原生驱动。**


持续更新中...

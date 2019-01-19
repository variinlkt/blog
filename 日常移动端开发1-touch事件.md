# 日常移动端开发1-touch事件

## 遇到的bug
之前利用了touch事件封装了一个简易的slider组件，实现滑动动画和carousel，但是后来发现在快速滑动的时候会在touchend的handler执行完后再执行一遍touchmove，在移动端尤其是安卓端这个bug经常被触发

一开始以为是通用bug，结果google了一下没几个人有这个问题，大部分是因为手指移出屏幕外而无法触发touchend事件
最后发现是因为自己在touchmove的handler里面加了requestAnimationFrame这个节流函数，它会让浏览器在**下一次重绘之前**调用指定函数来更新动画

也就是说，当我们在快速滑动的时候，触发了touchend事件后，还有一帧的touchmove没来得及被requestAnimationFrame执行，导致最后无法触发切换页面的动画过渡效果

坑爹啊。。。
## 关于touch事件的延伸
### click和touch的优先级

之前为了修改上面的bug在touchstart中调用了e.preventDefault(), 导致在menu组件添加的点击事件无法触发了

之前一直理解的click和touch的执行顺序是：

touchstart -> touchmove -> touchend -> click

google了才知道touch事件会被优先处理，touch事件经过捕获、处理、冒泡以后才会触发click事件

因此**touch阶段可以通过e.preventDefault()来取消系统触发的click事件**

### 移动端的click延迟300ms的原因

因为移动端有双击放大的事件，所以浏览器需要预留300ms再触发click事件

#### 解决方法：

①禁用缩放，表明这个页面是不可缩放的，那双击缩放的功能就没有意义了，此时浏览器可以禁用默认的双击缩放行为并且去掉300ms的点击延迟。

```
<meta name="viewport" content="user-scalable=no">
```
② `touch-action`这个CSS属性。这个属性指定了相应元素上能够触发的用户代理（也就是浏览器）的默认行为。如果将该属性值设置为`touch-action: none`，那么表示在该元素上的操作不会触发用户代理的任何默认行为，就无需进行300ms的延迟判断。

③通过监听touch事件来封装tap：

在touchstart、touchend时记录时间、手指位置，在touchend时进行比较，如果手指位置为同一位置（或允许移动一个非常小的位移值）且时间间隔较短（一般认为是200ms），且过程中未曾触发过touchmove，即可认为触发了“tap”



### 点透情况的发生

发生场景：A覆盖在B元素上，AB事件同时绑定了click事件，其中Aclick handler是点击A后A元素消失，点击a后会触发b的handler

这是因为在移动端浏览器，事件执行的顺序是

touchstart > touchend > click

而click事件有300ms的延迟，当touchstart事件把B元素隐藏之后，隔了300ms，浏览器触发了click事件，但是此时B元素不见了，所以该事件被派发到了A元素身上。如果A元素是一个链接，那此时页面就会意外地跳转。

#### 解决方法：

①使用touch事件代替click事件

②touchend设置e.preventDefault()，阻止click事件
### 文末分享
- 测试手机touch是否正常的地址：

![](https://dfiles.tita.com/Portal/110006/e3bda4dee47744a3ae055fb52a2c4f44.png)

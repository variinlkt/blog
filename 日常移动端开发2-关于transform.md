# 日常移动端开发2-关于transform
啊其实这跟移动端开发没啥关系，就是利用gpu加速对移动端的滑动动画优化了一下，感觉海星，不过还没测兼容性，因为我没有那么多测试机，先等测试提bug好了。

在oppo r9s Android 6.0.1里面没有卡顿的现象或bug

但是遇到了一些值得记录的东西，所以还是想让它拥有姓名——写篇博客记录一下：

用CSS3动画替代JS模拟动画的好处：

- 不占用JS主线程；
- 可以利用硬件加速；
- 浏览器可对动画做优化（元素不可见时不动画减少对FPS影响）

当我们用transform属性来改变元素时不会触发repaint，因为transform 动画由GPU控制，支持硬件加速，并不需要软件方面的渲染。

> 浏览器接收到页面文档后，会将文档中的标记语言解析为DOM树。
>
>DOM树和CSS结合后形成浏览器构建页面的渲染树。渲染树中包含了大量的渲染元素，每一个渲染元素会被分到一个图层中，每个图层又会被加载到GPU形成渲染纹理，而图层在GPU中 transform 是不会触发 repaint 的，这一点非常类似3D绘图功能，最终这些使用 transform 的图层都会由独立的合成器进程进行处理。

CSS transform **创建了一个新的复合图层，可以被GPU直接用来执行 transform 操作。**

当我们通过transform:translate去触发2D操作的时候其实也会触发repaint，只是**在动画的开始和结尾会触发**

过程：动画开始时，++生成新的复合图层++并加载为GPU的纹理用于初始化 repaint。然后由++GPU的复合器操纵整个动画的执行++。最后当动画结束时，再次++执行 repaint 操作删除复合图层++。



#### 如何避免动画前后的两次repaint？


```
transform: translateZ(0);
```

或

```
transform: rotateZ(360deg);
```

作用就是让浏览器执行 3D transform。浏览器通过该样式创建了一个独立图层，图层中的动画则有GPU进行预处理并且触发了硬件加速。

#### 浏览器什么时候会创建一个独立的复合图层呢？
- 3D 或者 CSS transform
- `<video>` 和 `<canvas>` 标签
- CSS filters
- 元素覆盖时，比如使用了 z-index 属性


#### 触发GPU的硬件加速的css属性有：

- transform
- opacity
- filter


#### gpu加速可能导致的问题：

1、通过-webkit-transform:transition3d/translateZ开启GPU硬件加速之后，有些时候可能会导致浏览器频繁闪烁或抖动，可以尝试以下办法解决：


```
-webkit-backface-visibility:hidden;
-webkit-perspective:1000;
```

2、gpu加速导致的布局问题：

一开始我把标识页面数量的小方块的html写进滑动组件的ul里面了，用的是`position：absolute`绝对定位，然后在滑动组件容器外使用了`position：relative`，在开启gpu加速后小方块居然以设置了transform的ul元素进行定位了

#### 经过多次试验后发现transform会导致其子元素的position定位失效，表现为：

##### ①子元素为absolute的，会将设置了transform的父元素作为标准进行定位

##### ②子元素为fixed的，定位会失效，变成absolute一样的行为表现，并随页面滚动而滚动

google了一下又发现：

##### ③已知：overflow不为visible的元素下有一个absolute的子元素，同时该元素到此子元素之间的嵌套元素中没有position为非static的元素，那么该子元素不会受到overflow的影响，即比如overflow hidden的父元素里面有个absolute的子元素，子元素如果溢出了也会显示出来

当我们给overflow元素加一个transform属性时，原本不受影响的子元素会变“听话”，乖乖隐藏溢出的部分

##### ④已知：当设置absolute元素的宽度为100%时，元素会参照第一个非static值的position元素计算，没有的话就按window来计算

当我们设置了transform属性，该元素里面的absolute子元素会按该元素宽度作为参照

##### ⑤transform属性会对层叠顺序造成影响，因为会触发创建层叠上下文


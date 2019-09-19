# scrollIntoView & scrollIntoViewIfNeeded & scroll-behavior
上周接触了移动端的键盘问题，从ios浏览器中第三方键盘弹出遮挡输入框的解决办法中，了解到了scrollIntoView这个函数。查了下还是蛮有意思的。

## 使用场景
- 滚动动画过渡

我们在用锚点定位时，点击后跳转的过程非常生硬，利用scroll-behavior可以快速实现滚动动画

- 解决ios浏览器中第三方键盘弹出遮挡输入框的问题

在iOS 中，如果输入框固定在页面底部，当键盘弹出时会将回复框遮盖部分，如图：

![](https://github.com/variinlkt/blog/blob/master/imgs/9fc15963-71b0-4a20-b678-ee8e541df621.jpg)
- 解决键盘遮挡输入框的问题
 
在安卓端，键盘弹出后，如果键盘高度高于输入框，会遮挡输入框，利用scrollIntoView可以一行代码就让输入框呈现于视口中。

### api/区别/兼容性
#### scrollIntoView
Element.scrollIntoView()方法让当前的元素滚动到浏览器窗口的可视区域内。
##### api

```
element.scrollIntoView(); // 等同于element.scrollIntoView(true) 
element.scrollIntoView(alignToTop); // Boolean型参数 
element.scrollIntoView(scrollIntoViewOptions); // Object型参数


# alignToTop
一个Boolean值：
如果为true，元素的顶端将和其所在滚动区的可视区域的顶端对齐。相应的 scrollIntoViewOptions: {block: "start", inline: "nearest"}。这是这个参数的默认值。
如果为false，元素的底端将和其所在滚动区的可视区域的底端对齐。相应的scrollIntoViewOptions: {block: "end", inline: "nearest"}。
# scrollIntoViewOptions 可选 
一个带有选项的object：
{
    behavior: "auto"  | "instant" | "smooth",
    block:    "start" | "end",
}
# behavior 可选
定义动画过渡效果， "auto", "instant", 或 "smooth" 之一。默认为 "auto"。
# block 可选
定义垂直方向的对齐， "start", "center", "end", 或 "nearest"之一。默认为 "start"。
# inline 可选
定义水平方向的对齐， "start", "center", "end", 或 "nearest"之一。默认为 "nearest"。
```

##### 兼容性
https://caniuse.com/#search=scrollIntoView
#### scrollIntoViewIfNeeded
Element.scrollIntoViewIfNeeded（）方法也是用来将不在浏览器窗口的可见区域内的元素滚动到浏览器窗口的可见区域。但如果该元素已经在浏览器窗口的可见区域内，则不会发生滚动。此方法是标准的Element.scrollIntoView()方法的专有变体。
##### api

```
element.scrollIntoViewIfNeeded(opt_center);

opt_center
一个 Boolean 类型的值，默认为true:
如果为true，则元素将在其所在滚动区的可视区域中居中对齐。
如果为false，则元素将与其所在滚动区的可视区域最近的边缘对齐。 根据可见区域最靠近元素的哪个边缘，元素的顶部将与可见区域的顶部边缘对准，或者元素的底部边缘将与可见区域的底部边缘对准。
```

##### 兼容性
https://caniuse.com/#feat=scrollintoviewifneeded

兼容性并不好，是非标准特性，一种WebKit专有的方法
#### css： scroll-behavior

```
scroll-behavior:smooth | auto
```

##### 兼容性
https://caniuse.com/#feat=css-scroll-behavior

ios不支持

##### 兼容不同浏览器

```
if (typeof window.getComputedStyle(document.body).scrollBehavior == 'undefined') {
   // 传统的JS平滑滚动处理代码...
}
```

```
/**
     @description 页面垂直平滑滚动到指定滚动高度
     @author zhangxinxu(.com)
    */
    var scrollSmoothTo = function (position) {
        if (!window.requestAnimationFrame) {
            window.requestAnimationFrame = function(callback, element) {
                return setTimeout(callback, 17);
            };
        }
        // 当前滚动高度
        var scrollTop = document.documentElement.scrollTop || document.body.scrollTop;
        // 滚动step方法
        var step = function () {
            // 距离目标滚动距离
            var distance = position - scrollTop;
            // 目标滚动位置
            scrollTop = scrollTop + distance / 5;
            if (Math.abs(distance) < 1) {
                window.scrollTo(0, position);
            } else {
                window.scrollTo(0, scrollTop);
                requestAnimationFrame(step);
            }
        };
        step();
    };
```

## 参考文章
https://www.zhangxinxu.com/wordpress/2018/10/scroll-behavior-scrollintoview-%e5%b9%b3%e6%bb%91%e6%bb%9a%e5%8a%a8/#

https://juejin.im/post/59d74afe5188257e8267b03f

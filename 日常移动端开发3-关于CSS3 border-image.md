# 关于CSS3 border-image
最近遇到了一个渐变边框的需求，之前也确实很少用到渐变边框，上网查了下或许跟border-image有关，因此在这里做个总结

![](https://image-static.segmentfault.com/159/293/1592938783-59ab907eefa3f)

border-image是一个复合属性，包括了source、slice、width、outset、repeat5️个属性
### source
即图片资源地址，url(...)

如果只设置了border-image-source属性而其他属性使用缺省值，则边框素材不会被划分九宫格，而是将整个素材按照边框宽度缩放至合适尺寸后安放在边框四角。

当该属性的取值为 none，或 url 无效时，边框将使用 border-style 属性的值。
### slice
图片剪裁位置

语法：`border-image-slice: number|%|fill;`

此属性指定顶部，右，底部，左边缘的图像向内偏移，分为九个区域：四个角，四边和中间。**图像中间部分将被丢弃**（完全透明的处理），除非填写关键字。

当参数个数=1时, 分为以下三种情况：
- 没有单位，如：border-image:url(border.png) 27；
- 百分比值，如：border-image:url(border.png) 30%；
- fill，保留图像的中间部分，一旦使用了fill关键字，源图片的中心块将作为该元素的背景。

当参数个数>1时，可以有2~4个参数，代表上右下左四个方位的剪裁，符合CSS普遍的方位规则，如`border-image：url(border.png) 30% 35% 40% 30%；`

![](https://image-static.segmentfault.com/369/307/3693076631-59a8f8f572d9c)
### repeat
即重复性，取值有repeat（重复）、round（平铺）、space（加空白的平铺）和stretch（拉伸）。stretch是默认值。

round会凑整填充（进行了适度的拉伸）。

repeat不进行拉伸，不凑整。

space: 图片平铺以填充区域。如果区域无法用整片图片填满，在每部分之间会加入空隙以填满区域。注意，该属性值并非所有浏览器都支持。

参数0~2个：
- 当参数为0个时，使用默认值stretch，如：`border-image:url(border.png) 30% 40%;`就等同于`border-image:url(border.png) 30% 40% stretch stretch;`
- 当参数为1个时，则表示水平方向及垂直方向均使用此参数
- 当参数为2个时，则第一个参数表水平方向，第二个参数表示垂直方向
## width
边框的宽度，用来限制相应区域背景图的范围。

相应背景区域的图像会根据这个属性值进行缩放。然后再重复或平铺或拉伸。

当同时设置了border-image-width和border-width属性时，那么边框的实际宽度由border-width属性决定。

![](https://image-static.segmentfault.com/180/116/180116240-5642fc2632630)

在复合写法中应该位于 slice属性 和repeat属性中间 用“/”间隔
如：`border-image:url(border.png) 27 / 6rem / repeat;`

语法：`border-image-width: [ <length> | <percentage> | <number> | auto ]{1,4}`

- length 带 px, em, in … 单位的尺寸值
- percentage 百分比
- number 不带单位的数字；它表示 border-width 的倍数
- auto 使用 auto， border-image-width 将会使用 border-image-slice 的值
### outset
边框背景扩散,相当于把原来的贴图位置向外延伸。不能为负值
在复合写法中应该位于 border-image-width 后面，用“/”间隔
如：`border-image:url(border.png) 27 / 6rem / 1.5rem /repeat;`

语法：`border-image-outset: [ <length> | <number> ]{1,4}`

## 渐变边框的实现

```
.border{
    border: 1px solid;
    border-image: linear-gradient(to top right, #83ACFF, #7BD3FF) 1;
}
```
然鹅，需求上的渐变边框是有圆角的，但是如果使用了border-image就无法使用圆角，因为浏览器应用了border-image就不再应用border-style了

#### 解决方法

```
.border{
  position: relative;
    border: 4px solid transparent;
    border-radius: 16px;
    background: linear-gradient(orange, violet);
    background-clip: padding-box;
    padding: 10px;
}
.border::after{
    position: absolute;
    top: -4px; bottom: -4px;
    left: -4px; right: -4px;
    background: linear-gradient(red, blue);
    content: '';
    z-index: -1;
    border-radius: 16px;
}
```

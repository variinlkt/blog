# 关于vue css的scoped和/deep/
## scoped 原理

> When a <style> tag has the scoped attribute, its CSS will apply to elements of the current component only. This is similar to the style encapsulation found in Shadow DOM. It comes with some caveats, but doesn't require any polyfills. It is achieved by using PostCSS to transform the following:

```
<style scoped>
.example {
  color: red;
}
</style>

<template>
  <div class="example">hi</div>
</template>
Into the following:

<style>
.example[data-v-f3f3eg9] {
  color: red;
}
</style>

<template>
  <div class="example" data-v-f3f3eg9>hi</div>
</template>
```

即`.a{}`选择器加了scope的样式会被编译为`.a[data-xxxx]`，而data-xxxx是scoped组件在html结构上添加的自定义属性，这样就能实现样式作用域了

## /deep/是什么
举个栗子，在a组件嵌套b组件，并且希望通过在a组件的css里面去控制b组件的样式

```
<style scoped>
.a .b {
	color:red;
}
</style>
```
这样实现是不行的，因为上面的代码会编译为：

```
.a[data-aaaaa] .b[data-aaaaa] {
	color:red;
}
```
但是b样式是在data-bbbbb属性的b组件里的，自然就改不了样式了

### 解决方法
#### /deep/
如果你希望scoped样式中的一个选择器能够作用得“更深”，例如影响子组件，你可以使用 /deep/ 操作符：

```

<style lang="less" scoped>
.text-box {
  /deep/ input {
    width: 166px;
    text-align: center;
  }
}
</style>
```
这个主要用于Sass、Less预处理器无法解析>>>的情况
#### >>>
对于上面这种写法，谷歌浏览器可能不支持：

![](https://github.com/variinlkt/blog/blob/master/imgs/20180925141317240.png)


```
.a /deep/ .b {
    color: red;
}
```
这样编译出来的css样式：

```
.a[data-v-f3f3eg9] .b { /* ... */ }
```

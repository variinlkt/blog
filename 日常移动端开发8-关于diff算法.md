# 关于diff算法


## 未优化的diff算法
![](https://github.com/variinlkt/blog/blob/master/imgs/1792722120-57065fae74fa9_articlex.png)

传统 diff 算法的复杂度为 O(n^3)。传统 diff 算法通过循环递归对节点进行依次对比，效率低下，算法复杂度达到 O(n^3)，其中 n 是树中节点的总数。

## react diff
复杂度:O(n) 
### diff 策略
- Web UI 中 DOM 节点跨层级的移动操作特别少，可以忽略不计。

- 拥有相同类的两个组件将会生成相似的树形结构，拥有不同类的两个组件将会生成不同的树形结构。

- 对于同一层级的一组子节点，它们可以通过唯一 id 进行区分。

基于以上三个前提策略，React 分别对 tree diff、component diff 以及 element diff 进行算法优化，事实也证明这三个前提策略是合理且准确的，它保证了整体界面构建的性能。

#### tree diff
React 对树进行分层比较，两棵树只会对同一层次的节点进行比较。
当发现节点已经不存在，则该节点及其子节点会被完全删除掉，不会用于进一步的比较。**这样只需要对树进行一次遍历，便能完成整个 DOM 树的比较。**

由于 React 只会简单的考虑同层级节点的位置变换，而对于不同层级的节点，只有创建和删除操作。当根节点发现子节点中 A 消失了，就会直接销毁 A；当 D 发现多了一个子节点 A，则会创建新的 A（包括子节点）作为其子节点。

当出现节点跨层级移动时，并不会出现想象中的移动操作，而是以 A 为根节点的树被整个重新创建，这是一种影响 React 性能的操作，因此 React 官方建议不要进行 DOM 节点跨层级的操作。
##### 代替 DOM 节点跨层级的操作的方法
用css来隐藏节点


#### component diff
对于同类型组件，virtual DOM可能并没有发生任何变化，这时我们可以通过shouldCompoenentUpdate钩子来告诉该组件是否进行diff，从而提高大量的性能。

如果不是，则将该组件判断为 dirty component，从而替换整个组件下的所有子节点。


一旦 React 判断新旧组件 是不同类型的组件，就不会比较二者的结构，而是直接删除 component 旧组件，重新创建 新组件 以及其子节点。
#### element diff
React 提出优化策略：允许开发者对同一层级的同组子节点，添加唯一 key 进行区分

当节点处于同一层级时，React diff 提供了三种节点操作，分别为：INSERT_MARKUP（插入）、MOVE_EXISTING（移动）和 REMOVE_NODE（删除）。

新老集合进行 diff 差异化对比，通过 key 发现新老集合中的节点都是相同的节点，因此无需进行节点删除和创建，只需要将老集合中节点的位置进行移动，更新为新集合中节点的位置
##### 移位算法
（其实网上那么多文章讲这个移位算法都写挺好的，我就不再写一遍了...）

可参考：https://segmentfault.com/a/1190000010686582

###### 比较核心的内容：
- 先创建节点，后删除
- 如果有相同key的节点，作以下判断，只有当「旧节点的idx」小于「最近访问的最新index」时，才会移动节点：

```
if(child._mountIndex < lastIndex){
    return makeMove(child, afterNode, toIndex);
}
```

- 遍历节点时每次都会更新lastIndex：
```
lastIndex = Math.max(prevChild._mountIndex, lastIndex);
```
###### 导致的问题
如下图所示，若新集合的节点更新为：D、A、B、C，与老集合对比只有 D 节点移动，而 A、B、C 仍然保持原有的顺序，理论上 diff 应该只需对 D 执行移动操作，然而由于 D 在老集合的位置是最大的，导致其他节点的 _mountIndex < lastIndex，造成 D 没有执行移动操作，而是 A、B、C 全部移动到 D 节点后面的现象。

![](https://github.com/variinlkt/blog/blob/master/imgs/1b8dac5b9b3e4452dec8d5447d7717ad_r.jpg)

在开发过程中，尽量减少类似将最后一个节点移动到列表首部的操作，当节点数量过大或更新操作过于频繁时，在一定程度上会影响 React 的渲染性能。

### react diff总结
- tree diff：对树进行分层比较，两棵树只会对同一层次的节点进行比较
- component diff：
如果是同一类型的组件，按照原策略继续比较 virtual DOM tree。
如果不是，则将该组件判断为 dirty component，从而替换整个组件下的所有子节点。
对于同一类型的组件，有可能其 Virtual DOM 没有任何变化，如果能够确切的知道这点那可以节省大量的 diff 运算时间，因此 React 允许用户通过 shouldComponentUpdate() 来判断该组件是否需要进行 diff。
- element diff：当节点处于同一层级时，React diff 提供了三种节点操作，分别为：插入、移动、删除。

## vue diff
哎，既然都学了react的diff，这里就再总结一哈vue的diff吧

其实vue的diff跟react大同小异了
### vnode
先来看下一个vnode对象的属性：

```
// body下的 <div id="v" class="classA"><div> 对应的 oldVnode 就是

{
  el:  div  //对真实的节点的引用，本例中就是document.querySelector('#id.classA')
  tagName: 'DIV',   //节点的标签
  sel: 'div#v.classA'  //节点的选择器
  data: null,       // 一个存储节点属性的对象，对应节点的el[prop]属性，例如onclick , style
  children: [], //存储子节点的数组，每个子节点也是vnode结构
  text: null,    //如果是文本节点，对应文本节点的textContent，否则为null
}
```
### patch
diff的过程就是调用patch函数，就像打补丁一样修改真实dom。
```
var oldVnode = patch (oldVnode, vnode);
function patch (oldVnode, vnode) {//节点值得比较时
    if (sameVnode(oldVnode, vnode)) {
        patchVnode(oldVnode, vnode)
    } else {//节点不值得比较
        const oEl = oldVnode.el
        let parentEle = api.parentNode(oEl)
        createEle(vnode)//创建真实dom
        if (parentEle !== null) {
            api.insertBefore(parentEle, vnode.el, api.nextSibling(oEl))
            api.removeChild(parentEle, oldVnode.el)
            oldVnode = null
        }//新节点直接把老节点整个替换了
    }
    return vnode
}

function sameVnode(oldVnode, vnode){//key和sel相等表示节点值得比较
    return vnode.key === oldVnode.key && vnode.sel === oldVnode.sel
}

function patchVnode (oldVnode, vnode) {
    const el = vnode.el = oldVnode.el//vnode.el引用到现在的真实dom
    let i, oldCh = oldVnode.children, ch = vnode.children
    if (oldVnode === vnode) return//没有变化
    if (oldVnode.text !== null && vnode.text !== null && oldVnode.text !== vnode.text) {//文本节点不同
        api.setTextContent(el, vnode.text)
    }else {//文本节点相同
        updateEle(el, vnode, oldVnode)//比较子节点
        if (oldCh && ch && oldCh !== ch) {//两个节点都有子节点，而且它们不一样
            updateChildren(el, oldCh, ch)
        }else if (ch){//只有新的节点有子节点
            createEle(vnode) //创建子节点
        }else if (oldCh){//只有旧的节点有子节点
            api.removeChildren(el)//删除子节点
        }
    }
}
```
#### vnode和进入patch之前的不同在哪？
唯一的改变就是之前vnode.el = null,而现在它引用的是对应的真实dom。

### updateChildren
![](https://github.com/variinlkt/blog/blob/master/imgs/3332891351-58d13c20889f8_articlex.png)

代码配合上图食用更佳
```

updateChildren (parentElm, oldCh, newCh) {
    let oldStartIdx = 0, newStartIdx = 0
    let oldEndIdx = oldCh.length - 1
    let oldStartVnode = oldCh[0]
    let oldEndVnode = oldCh[oldEndIdx]
    let newEndIdx = newCh.length - 1
    let newStartVnode = newCh[0]
    let newEndVnode = newCh[newEndIdx]
    let oldKeyToIdx
    let idxInOld
    let elmToMove
    let before
    while (oldStartIdx <= oldEndIdx && newStartIdx <= newEndIdx) {
            if (oldStartVnode == null) {   //对于vnode.key的比较，会把oldVnode = null
                oldStartVnode = oldCh[++oldStartIdx] 
            }else if (oldEndVnode == null) {
                oldEndVnode = oldCh[--oldEndIdx]
            }else if (newStartVnode == null) {
                newStartVnode = newCh[++newStartIdx]
            }else if (newEndVnode == null) {
                newEndVnode = newCh[--newEndIdx]
            }else if (sameVnode(oldStartVnode, newStartVnode)) {
            //头头比较
                patchVnode(oldStartVnode, newStartVnode)
                oldStartVnode = oldCh[++oldStartIdx]
                newStartVnode = newCh[++newStartIdx]
            }else if (sameVnode(oldEndVnode, newEndVnode)) {
            //尾尾比较
                patchVnode(oldEndVnode, newEndVnode)
                oldEndVnode = oldCh[--oldEndIdx]
                newEndVnode = newCh[--newEndIdx]
            }else if (sameVnode(oldStartVnode, newEndVnode)) {
            //头尾比较
                patchVnode(oldStartVnode, newEndVnode)
                api.insertBefore(parentElm, oldStartVnode.el, api.nextSibling(oldEndVnode.el))
                oldStartVnode = oldCh[++oldStartIdx]
                newEndVnode = newCh[--newEndIdx]
            }else if (sameVnode(oldEndVnode, newStartVnode)) {
            //尾头比较
                patchVnode(oldEndVnode, newStartVnode)
                api.insertBefore(parentElm, oldEndVnode.el, oldStartVnode.el)
                oldEndVnode = oldCh[--oldEndIdx]
                newStartVnode = newCh[++newStartIdx]
            }else {
               // 使用key时的比较
                if (oldKeyToIdx === undefined) {
                    oldKeyToIdx = createKeyToOldIdx(oldCh, oldStartIdx, oldEndIdx) // 有key就生成index表
                }
                idxInOld = oldKeyToIdx[newStartVnode.key]//在index表里面找旧节点的idx
                if (!idxInOld) {//找不到，说明是新添加的节点
                    api.insertBefore(parentElm, createEle(newStartVnode).el, oldStartVnode.el)
                    newStartVnode = newCh[++newStartIdx]
                }
                else {//移动旧节点
                    elmToMove = oldCh[idxInOld]
                    if (elmToMove.sel !== newStartVnode.sel) {
                        api.insertBefore(parentElm, createEle(newStartVnode).el, oldStartVnode.el)
                    }else {
                        patchVnode(elmToMove, newStartVnode)
                        oldCh[idxInOld] = null
                        api.insertBefore(parentElm, elmToMove.el, oldStartVnode.el)
                    }
                    newStartVnode = newCh[++newStartIdx]
                }
            }
        }
        if (oldStartIdx > oldEndIdx) {
            before = newCh[newEndIdx + 1] == null ? null : newCh[newEndIdx + 1].el
            addVnodes(parentElm, before, newCh, newStartIdx, newEndIdx)
        }else if (newStartIdx > newEndIdx) {
            removeVnodes(parentElm, oldCh, oldStartIdx, oldEndIdx)
        }
}
```
#### vue diff 小结
- 只会在同层级比较，不会跨层级比较
- 头头/尾尾节点比较
- 头尾/尾头节点比较
- 设key后，除了头尾两端的比较外，还会从用key生成的对象oldKeyToIdx中查找匹配的节点，所以为节点设置key可以更高效的利用dom。


## 参考文章
https://zhuanlan.zhihu.com/p/20346379?refer=purerender

https://segmentfault.com/a/1190000004913592?utm_source=tag-newest

https://segmentfault.com/a/1190000010686582

https://segmentfault.com/a/1190000008782928

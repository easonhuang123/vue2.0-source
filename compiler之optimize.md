## optimizer

回顾一下上一部分的内容，我们从template一步步的转化为一个抽象语法树AST，而今天这部分比较简单，就是compiler中的第二步——对AST进行静态优化。其中主要操作就是找出AST中静态内容，并对其添加标注，为后面转化成render代码做准备。

静态内容指的就是不需要进行多次刷新的内容，我们就把它当做常量处理，在重新渲染的时候直接跳过，无需特意改变它们。

### src/compiler/optimizer.js

打开这个文件看一下也就一百多行代码，很容易理解~ 我们赶紧开始学习吧

```
export function optimize (root: ?ASTElement, options: CompilerOptions) {
  if (!root) return
  isStaticKey = genStaticKeysCached(options.staticKeys || '')
  isPlatformReservedTag = options.isReservedTag || no

  // first pass: mark all non-static nodes. 标记所有非静态节点
  markStatic(root)
  // second pass: mark static roots. 标记静态子树
  markStaticRoots(root, false)
}
```

从`optimize`函数看起，先是判断传入的keys是否在`type,tag,attrsList,attrsMap,plain,parent,children,attrs,staticKeys`这些属性限定范围内，其中`staticKeys`指的就是`staticClass`静态类, `staticStyle`静态样式。
```
function genStaticKeys (keys: string): Function {
  return makeMap(
    'type,tag,attrsList,attrsMap,plain,parent,children,attrs' +
    (keys ? ',' + keys : '')
  )
}
```
然后是判断是否是平台保留标签`isPlatformReservedTag`，这里所有的HTML和SVG标签都会返回true。

接着就是我们的主菜了，分为两个步骤：

- 标记静态节点标签，区分非静态节点和静态节点

- 标记静态根节点

```
function markStatic (node: ASTNode) {
  node.static = isStatic(node)
  if (node.type === 1) {
    // 假如有slot的话不能设为静态
    if (
      // 不是保留标签
      !isPlatformReservedTag(node.tag) &&
      // 不是slot
      node.tag !== 'slot' &&
      // 不是一个内联模板容器
      node.attrsMap['inline-template'] == null
    ) {
      return
    }
    for (let i = 0, l = node.children.length; i < l; i++) {
      const child = node.children[i]
      markStatic(child)
      // 只要有一个子树不是静态的话整个都不是静态
      if (!child.static) {
        node.static = false
      }
    }
    if (node.ifConditions) {
      for (let i = 1, l = node.ifConditions.length; i < l; i++) {
        const block = node.ifConditions[i].block
        markStatic(block)
        if (!block.static) {
          node.static = false
        }
      }
    }
  }
}
```
第一步`markStatic`，我们先判断该节点是否是静态的，在`isStatic`中判断出type为2或3的节点还有一些特殊的节点。我在注释中有所标记。
```
function isStatic (node: ASTNode): boolean {
  if (node.type === 2) { // expression表达式不是静态文本
    return false
  }
  if (node.type === 3) { // text是静态文本
    return true
  }
  return !!(node.pre || ( // v-pre指令,结点的子内容是不做编译
    !node.hasBindings && // 无动态绑定
    !node.if && !node.for && // 无 v-if or v-for or v-else
    !isBuiltInTag(node.tag) && // not a built-in 不是内置的标签，内置的标签有slot和component
    isPlatformReservedTag(node.tag) && // not a component 是平台保留标签
    !isDirectChildOfTemplateFor(node) && // 不是template标签的直接子元素且没有包含在for循环中
    Object.keys(node).every(isStaticKey) // 结点包含的属性只能有isStaticKey中指定的几个
  ))
}
```
然后再对type为1的节点进行递归操作，只要有一个它的子树不是静态的话它就不是静态的。

第二步`markStaticRoots`
```
function markStaticRoots (node: ASTNode, isInFor: boolean) {
  if (node.type === 1) {
    if (node.static || node.once) {
      node.staticInFor = isInFor
    }
    // 对于一个静态根结点，它不应该只包含静态文本，否则消耗会超过获得的收益，更好的做法让它每次渲染时都刷新
    if (node.static && node.children.length && !(
      node.children.length === 1 &&
      node.children[0].type === 3
    )) {
      node.staticRoot = true
      return
    } else {
      node.staticRoot = false
    }
    if (node.children) {
      for (let i = 0, l = node.children.length; i < l; i++) {
        markStaticRoots(node.children[i], isInFor || !!node.for)
      }
    }
    if (node.ifConditions) {
      for (let i = 1, l = node.ifConditions.length; i < l; i++) {
        markStaticRoots(node.ifConditions[i].block, isInFor)
      }
    }
  }
}
```
注意，当这个静态节点只有静态文本的时候我们设为false，因为如果只包含静态文本的话渲染过程的消耗会大于获得的收益，更好的做法让它每次渲染时都刷新。然后也是对其子树进行递归操作，添加上`staticRoot`属性。

我们再看一眼AST的结构（其实上一篇文章中未优化的AST是没有`static`和`staticRoot`属性的）：

![AST](./images/ast.png)

优化AST这部分就这样愉快的结束啦~ 感谢你的阅读 : )
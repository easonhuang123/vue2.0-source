## codegen

这部分是compiler的最后一步，将AST转化为render字符串，对应文件是`src/compiler/codegen/index.js`，然而你打开文件想预览一下整体结构，发现鼠标滚轮滚了几轮都还没到底 [黑人问号脸] ,因为代码有五百多行 [微笑脸] 。咬咬牙还是把它看完了，其实你会发现这代码并不难理解，就是要兼顾的内容，涉及的范围太大了，毕竟vue是个这么庞大的框架，所以。。。。所以还是不能放弃！本篇我就不啰嗦啦，只说一些我觉得有必要分享的地方，大家可以自己慢慢品尝~ ~ ~ ~ 

### 开始之前

这部分的学习方法还是一样，善用断点。我们先拿回上次说到的小栗子，输出一下它由AST转化来的render字符串
![](./images/render.png)
图片太长，我稍微分段了一下，长这样
```
render :
    "with(this){return _c('div',{staticClass:"div"},
        [
            _v("\n    我是最普通<的文本\n      "),
            _c('p',[_v(_s(msg))]),_v(" "),
            _c('a',{attrs:{"href":src}},[_v("这是我的简书链接")])
        ]
    )}"
```
看完玩意儿之后觉得糊里糊涂的，有很多奇奇怪怪的指令，所以我先提前解释一下各个指令的含义，方便等一下我们学习时可以参考~

_c createElement方法，创建一个元素，它的第一个参数是要定义的元素标签名、第二个参数是元素上添加的属性，第三个参数是子元素数组，第四个参数是子元素数组进行归一化处理的级别

_v 本文节点

_s 需解析的文本，之前在parser阶段已经有所修饰

_m 渲染静态内容

_o v-once静态组件

_l v-for节点

_e 注释节点

_t slot节点

接下来就可以开始看代码咯

### src/compiler/codegen/index.js

我们从入口函数出发`generate`
```
export function generate (
  ast: ASTElement | void,
  options: CompilerOptions
): CodegenResult {
  const state = new CodegenState(options)
  const code = ast ? genElement(ast, state) : '_c("div")'
  return {
    render: `with(this){return ${code}}`,
    staticRenderFns: state.staticRenderFns
  }
}
```
喵喵喵？这也太棒了吧，只有十几行。先是定义一个对象`CodegenState`，里面包含转化时需要的属性和方法，我们看看
```
export class CodegenState {
  options: CompilerOptions;
  warn: Function;
  transforms: Array<TransformFunction>;
  dataGenFns: Array<DataGenFunction>;
  directives: { [key: string]: DirectiveFunction };
  maybeComponent: (el: ASTElement) => boolean;
  onceId: number;
  staticRenderFns: Array<string>;

  constructor (options: CompilerOptions) {
    this.options = options
    this.warn = options.warn || baseWarn
    this.transforms = pluckModuleFunction(options.modules, 'transformCode')
    this.dataGenFns = pluckModuleFunction(options.modules, 'genData')
    this.directives = extend(extend({}, baseDirectives), options.directives)
    const isReservedTag = options.isReservedTag || no
    this.maybeComponent = (el: ASTElement) => !isReservedTag(el.tag)
    this.onceId = 0
    this.staticRenderFns = []
  }
}
```
`transforms`我输出了一下是个空数组，我们先忽略；`dataGenFns`是对静态类和静态样式的处理，`directives`是对指令的相关操作，我们以后会再说；`isReservedTag`保留标签标志；`maybeComponent`字面意思不是保留标签就可能是组件；`onceId`使用`v-once`的递增id；`staticRenderFns`是对静态根节点的处理。

继续往下看，如果AST为空，`code = '_c("div")'`，即创建一个空的div，否则执行`code = genElement(ast)`。

```
export function genElement (el: ASTElement, state: CodegenState): string {
  if (el.staticRoot && !el.staticProcessed) {
    return genStatic(el, state)
  } else if (el.once && !el.onceProcessed) {
    return genOnce(el, state)
  } else if (el.for && !el.forProcessed) {
    return genFor(el, state)
  } else if (el.if && !el.ifProcessed) {
    return genIf(el, state)
  } else if (el.tag === 'template' && !el.slotTarget) {
    return genChildren(el, state) || 'void 0'
  } else if (el.tag === 'slot') {
    return genSlot(el, state)
  } else {
    let code
    if (el.component) {
      code = genComponent(el.component, el, state)
    } else {
      const data = el.plain ? undefined : genData(el, state)

      const children = el.inlineTemplate ? null : genChildren(el, state, true)
      code = `_c('${el.tag}'${
        data ? `,${data}` : '' // data
      }${
        children ? `,${children}` : '' // children
      })`
    }
    // 静态class，style 处理
    for (let i = 0; i < state.transforms.length; i++) {
      code = state.transforms[i](el, code)
    }
    return code
  }
}
```
如果该节点是个静态子树，我们就执行`genStatic`对其进行处理。
```
function genStatic(el: ASTElement, state: CodegenState, once: ?boolean): string {
  el.staticProcessed = true
  state.staticRenderFns.push(`with(this){return ${genElement(el, state)}}`)
  return `_m(${
    state.staticRenderFns.length - 1
  },${
    el.staticInFor ? 'true' : 'false'
  },${
    once ? 'true' : 'false'
  })`
}
```
添加`staticProcessed`属性，插入`staticRenderFns`子树中，再执行回`genElement`，返回提取静态子树`_m`结构的字符串。

如果该节点有`v-once`标签，执行`genOnce`
```
function genOnce (el: ASTElement, state: CodegenState): string {
  el.onceProcessed = true
  if (el.if && !el.ifProcessed) {
    return genIf(el, state)
  } else if (el.staticInFor) {
    let key = ''
    let parent = el.parent
    while (parent) {
      if (parent.for) {
        key = parent.key
        break
      }
      parent = parent.parent
    }
    if (!key) {
      return genElement(el, state)
    }
    return `_o(${genElement(el, state)},${state.onceId++},${key})`
  } else {
    return genStatic(el, state, true)
  }
}
```
添加`onceProcessed`属性，里面分了if和for的情况，前者执行`genIf`，后者向上查找key，找到的话返回静态内容`_o`结构的字符串，假如两种情况都不是的话则像静态子树一样处理。

`genFor`和`genIf`的处理我们先忽略。

如果该节点是template，则执行`genChildren`，这个函数是对子节点归一化处理的级别标记，后面还会涉及，放到后面再说。

如果该节点是slot，则执行`genSlot`,返回静态内容`_t`结构的字符串。
```
function genSlot (el: ASTElement, state: CodegenState): string {
  const slotName = el.slotName || '"default"'
  const children = genChildren(el, state)
  let res = `_t(${slotName}${children ? `,${children}` : ''}`
  const attrs = el.attrs && `{${el.attrs.map(a => `${camelize(a.name)}:${a.value}`).join(',')}}`
  const bind = el.attrsMap['v-bind']
  if ((attrs || bind) && !children) {
    res += `,null`
  }
  if (attrs) {
    res += `,${attrs}`
  }
  if (bind) {
    res += `${attrs ? '' : ',null'},${bind}`
  }
  return res + ')'
}
```

假如以上条件都不是。则这就是个普通的节点了，先判断它是否有子组件，执行`genComponent`，否则的话执行普通节点的步骤，验证是否是空节点，执行`genData`，再执行`genChildren`进行子节点归一化处理的级别标记。我们也忽略有子组件的可能，直接看最普通的做法。
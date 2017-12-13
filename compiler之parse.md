## 开始之前

本篇讲述的是compiler的第一步——parse，就是将模板template转化为AST的这一步，在开始之前先做一些准备工作，方便等一下我们学习的时候可以参考

首先是上一篇提到的在web平台上的`baseOptions`，我们再次拿出来，因为里面的属性可能会在之后的内容多次出现。
```
{
  expectHTML: true, // 是否期望HTML，不知道是啥反正web中的是true
  modules, // klass和style，对模板中类和样式的解析
  directives, // v-model、v-html、v-text
  isPreTag, // v-pre标签
  isUnaryTag, // 单标签，比如img、input、iframe
  mustUseProp, // 需要使用props绑定的属性，比如value、selected等
  canBeLeftOpenTag, // 可以不闭合的标签，比如tr、td等
  isReservedTag, // 是否是保留标签，html标签和SVG标签
  getTagNamespace, // 命名空间，svg和math
  staticKeys: genStaticKeys(modules) // staticClass,staticStyle。
}
```
然后我们这部分主要学习`src/compiler/parser/index.js` ， `src/compiler/parser/html-parser.js` 和 `src/compiler/parser/text-parser.js`，里面有一些正则表达式我们先提前在这里展示一下，方便我们之后的学习~ emmm。。。其实我一看到正则就晕，但是还是坚持把它翻译完了，假如有错误之处请及时指出，谢谢~

### src/compiler/parser/index.js

```
// @和v-on 事件绑定
const onRE = /^@|^v-on:/

// v-xxx, @, : 事件/数据绑定
const dirRE = /^v-|^@|^:/

// v-for 中的属性值，如 (item, index) of items
const forAliasRE = /(.*?)\s+(?:in|of)\s+(.*)/

// v-for 中的前部分，如 (item, index)
const forIteratorRE = /\((\{[^}]*\}|[^,]*),([^,]*)(?:,([^,]*))?\)/

// : 绑定属性
const argRE = /:(.*)$/

// 以:或者v-bind: 开头的属性  
const bindRE = /^:|^v-bind:/

// .修饰符
const modifierRE = /\.[^.]+/g
```
### src/compiler/parser/html-parser.js
```
// 标准命名规范
const ncname = '[a-zA-Z_][\\w\\-\\.]*'

// 捕获整体内容
const qnameCapture = `((?:${ncname}\\:)?${ncname})`

// 开始标签开头
const startTagOpen = new RegExp(`^<${qnameCapture}`)

// 开始标签结尾
const startTagClose = /^\s*(\/?)>/

// 结束标签
const endTag = new RegExp(`^<\\/${qnameCapture}[^>]*>`)

// DOCTYPE
const doctype = /^<!DOCTYPE [^>]+>/i

// 注释 <!--
const comment = /^<!--/

// <![CDATA
const conditionalComment = /^<!\[/
```
### src/compiler/parser/text-parser.js

```
// 默认分隔符{{ }}
const defaultTagRE = /\{\{((?:.|\n)+?)\}\}/g

// 判断特殊字符作为分隔符
const regexEscapeRE = /[-.*+?^${}()|[\]\/\\]/g
```

## 我们来举个栗子吧

有的同学（像我一样）可能一开始看到每个文件都几百几百行的马上被吓跪了，别怕！慢慢看慢慢理解，一定可以的，而且会吸收得越来越快！为了照顾小白们（我自己），我就举个最简单的栗子来看看template是怎么一步步转化成AST的吧~
```
const vue = new Vue({
    el: '#app',
    template: `
    <div class='div'>
        我是最普通的文本
        <p>{{msg}}</p>
        <a :href='src'>这是我的简书链接</a>
    </div>`,
    data: {
        msg: '我是eason',
        src: 'http://www.jianshu.com/u/2a330ce15dd5'
    }
})
```
### src/compiler/parser/index.js

我们先看入口函数`parse`，一开始是定义一些属性，还有对传入选项中一些属性进行保存和访问，由于我们的例子比较简单，这些可以先略过。

接着是一个超长函数`parseHTML`，认真一下原来它是来自`html-parser.js`，一长串的是传入的参数，参数中除了一些标签之外还有几个重要的函数，start，end，chars 和 comment，他们的功能分别是对元素开始时的处理，对元素结束时的处理，对文本的处理，对注释的处理。具体实现的话我们结合`html-parser.js`中调用时再进行分析。

### src/compiler/parser/html-parser.js
直接看函数`parseHTML`，定义了一个栈用来存放元素，还有一些标签（先忽略，用到再看），接下来就是一个`while(html)`的循环操作，大家应该能猜到这操作就是不断地处理模板，每一部分处理完后切割剩余的进入循环继续处理。其中先提一下`advance`函数，它是负责将索引`index`向前移动，指向未处理模板的开始位置，进行模板切割。
```
function advance (n) {
    index += n
    html = html.substring(n)
}
```
我们进入循环，继续分析。我们先进入`textEnd === 0`第一个条件中
```
// 过滤掉注释<!-- -->，从‘-->’后开始读取
if (comment.test(html)) {
    const commentEnd = html.indexOf('-->')

    if (commentEnd >= 0) {
    if (options.shouldKeepComment) {
        options.comment(html.substring(4, commentEnd))
    }
    // 3 是 ‘-->’的长度
    advance(commentEnd + 3)
    continue
    }
}

// 过滤<![   ]>注释的内容
if (conditionalComment.test(html)) {
    const conditionalEnd = html.indexOf(']>')

    if (conditionalEnd >= 0) {
    advance(conditionalEnd + 2)
    continue
    }
}

// 过滤Doctype:
const doctypeMatch = html.match(doctype)
if (doctypeMatch) {
    advance(doctypeMatch[0].length)
    continue
}

// 结束标签处理
const endTagMatch = html.match(endTag)
if (endTagMatch) {
    const curIndex = index
    advance(endTagMatch[0].length)
    parseEndTag(endTagMatch[1], curIndex, index)
    continue
}

// 开始标签处理
const startTagMatch = parseStartTag()
if (startTagMatch) {
    // 处理startTagMatch
    handleStartTag(startTagMatch)
    if (shouldIgnoreFirstNewline(lastTag, html)) {
    advance(1)
    }
    continue
}
```
先是对注释`<!-- -->`和`<![ ]>`，还有`Doctype`进行过滤，然后我们一开始没有`endTag`，所以没有结束标签的相关处理，接着是`startTagMatch`，我们看看函数`parseStartTag`

```
  function parseStartTag () {
    const start = html.match(startTagOpen)
    if (start) {
      const match = {
        tagName: start[1],
        attrs: [],
        start: index
      }
      advance(start[0].length)
      let end, attr
      while (!(end = html.match(startTagClose)) && (attr = html.match(attribute))) {
        advance(attr[0].length)
        match.attrs.push(attr)
      }
      if (end) {
        match.unarySlash = end[1]
        advance(end[0].length)
        match.end = index
        return match
      }
    }
  }
```
一开始我们讲到了正则`startTagOpen`是开始标签的开头，我们匹配到了div标签，接着是定义了一个变量match，我们输出一下看看结果：
```
match = {
    tagName: "div",
    attrs: [" class='div'", "class", "=", undefined, "div", undefined, index: 0, input: " class='div'>↵                我是最普通的<文本>↵         …   <a :href='src'>这是我的简书链接</a>↵            </div>"],
    start: 0,
    unarySlash: "",
    end: 17}
```
这个函数的大致操作就是读取开始标签中的属性值并用一个数组存起来，然后索引值跳到标签之后，处理完成。

然后回到原函数中，`handleStartTag`对刚才返回的结果`startTagMatch`进行处理，继续看看函数`handleStartTag`。

```
function handleStartTag (match) {
    const tagName = match.tagName
    const unarySlash = match.unarySlash

    if (expectHTML) {
      if (lastTag === 'p' && isNonPhrasingTag(tagName)) {
        parseEndTag(lastTag)
      }
      if (canBeLeftOpenTag(tagName) && lastTag === tagName) {
        parseEndTag(tagName)
      }
    }

    const unary = isUnaryTag(tagName) || !!unarySlash

    const l = match.attrs.length
    const attrs = new Array(l)
    for (let i = 0; i < l; i++) {
      const args = match.attrs[i]
      if (IS_REGEX_CAPTURING_BROKEN && args[0].indexOf('""') === -1) {
        if (args[3] === '') { delete args[3] }
        if (args[4] === '') { delete args[4] }
        if (args[5] === '') { delete args[5] }
      }
      const value = args[3] || args[4] || args[5] || ''
      const shouldDecodeNewlines = tagName === 'a' && args[1] === 'href'
        ? options.shouldDecodeNewlinesForHref
        : options.shouldDecodeNewlines
      attrs[i] = {
        name: args[1],
        value: decodeAttr(value, shouldDecodeNewlines)
      }
    }
    
    if (!unary) {
      stack.push({ tag: tagName, lowerCasedTag: tagName.toLowerCase(), attrs: attrs })
      lastTag = tagName
    }

    if (options.start) {
      options.start(tagName, attrs, unary, match.start, match.end)
    }
}
```

它的大致流程就是先循环遍历match中的属性数组attrs进行格式封装，接着是`if(!unary)`对于非单标签元素的话，将他们封装后再入栈，然后是进行执行`start`操作。此时输出一下attrs和stack你们看看
```
attrs = [
    {
        name: 'class',
        value: 'div'
    }
]
stack = [
    {
        tag: 'div',
        lowerCasedTag: 'div',
        attrs
    }
]
```
`start`操作是作为参数传入的，这次我们分析一下`start`。由于代码过长我就不贴代码了，我先说说整体的流程，然后你们对着源码一起看。

`start`整体流程：
- 定义初始的抽象语法树AST
- 对AST进行预处理
- 对vue的指令进行处理v-pre、v-if、v-for、v-once、slot、key、ref
- 对根节点进行处理
- 元素父子关系的绑定
- 将当前AST入栈
- 对AST进行后处理

指令部分我们日后再逐一分析，我们可以输出AST、堆栈或者一些结构比较复杂的变量，通过观察它们可以方便我们的理解，也不会太枯燥。




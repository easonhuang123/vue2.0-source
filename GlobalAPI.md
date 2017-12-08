## Global-api——绑定全局配置和API

回顾一下`src/core/index.js`文件
```
import Vue from './instance/index'
import { initGlobalAPI } from './global-api/index'
import { isServerRendering } from 'core/util/env'

initGlobalAPI(Vue)

Object.defineProperty(Vue.prototype, '$isServer', {
  get: isServerRendering
})

Object.defineProperty(Vue.prototype, '$ssrContext', {
  get () {
    /* istanbul ignore next */
    return this.$vnode && this.$vnode.ssrContext
  }
})

Vue.version = '__VERSION__'

export default Vue
```

我们注意到了`initGlobalAPI(Vue)`，说明我们在引用Vue的时候在这里给Vue初始化了一些全局的配置和API，我们再进到`global-api`看看具体定义了哪些配置和API。

### global-api/index.js
```
import config from '../config'
import { initUse } from './use'
import { initMixin } from './mixin'
import { initExtend } from './extend'
import { initAssetRegisters } from './assets'
import { set, del } from '../observer/index'
import { ASSET_TYPES } from 'shared/constants'
import builtInComponents from '../components/index'

import {
  warn,
  extend,
  nextTick,
  mergeOptions,
  defineReactive
} from '../util/index'

export function initGlobalAPI (Vue: GlobalAPI) {
  const configDef = {}
  configDef.get = () => config
  if (process.env.NODE_ENV !== 'production') {
    configDef.set = () => {
      warn(
        'Do not replace the Vue.config object, set individual fields instead.'
      )
    }
  }
  // 这里定义了Vue.config是一个对象，包含 Vue 的全局配置
  Object.defineProperty(Vue, 'config', configDef)

  // util方法尽量别碰
  Vue.util = {
    warn,
    extend,
    mergeOptions,
    defineReactive
  }

  // 绑定全局API——Vue.set，Vue.delete，Vue.nextTick
  Vue.set = set
  Vue.delete = del
  Vue.nextTick = nextTick

  Vue.options = Object.create(null)
  ASSET_TYPES.forEach(type => {
    Vue.options[type + 's'] = Object.create(null)
  })

  Vue.options._base = Vue

  extend(Vue.options.components, builtInComponents)

  // 初始化Vue.extend，Vue.mixin，Vue.extend
  // AssetRegisters就是component，directive，filter三者
  initUse(Vue)
  initMixin(Vue)
  initExtend(Vue)
  initAssetRegisters(Vue)
}
```
大概的理解都写在了注释里，都是一些我们平时熟知的一些API，感兴趣的话我们可以逐一去看，都不难理解。

### global-api/use.js
```
export function initUse (Vue: GlobalAPI) {
  Vue.use = function (plugin: Function | Object) {
    const installedPlugins = (this._installedPlugins || (this._installedPlugins = []))
    if (installedPlugins.indexOf(plugin) > -1) {
      return this
    }

    // additional parameters
    const args = toArray(arguments, 1)
    args.unshift(this)
    if (typeof plugin.install === 'function') {
      plugin.install.apply(plugin, args)
    } else if (typeof plugin === 'function') {
      plugin.apply(null, args)
    }
    installedPlugins.push(plugin)
    return this
  }
}
```

这个方法定义了`Vue.use`的使用，看懂了这个对我们之后直接写Vue插件很有帮助哦~这个方法为`Vue.use`定义了两种使用方法。第一种就是你写的插件（`plugin`参数）是一个对象，然后`Vue.use`会找到这个对象中的`install`方法作为入口调用，第二种就是你传来的参数就是一个函数，然后直接执行这个函数。当`install`方法被同一个插件多次调用，插件将只会被安装一次。

### global-api/mixin.js
```
export function initMixin (Vue: GlobalAPI) {
  Vue.mixin = function (mixin: Object) {
    this.options = mergeOptions(this.options, mixin)
    return this
  }
}
```
mixin没啥好说的，就是将你传来的参数对象和vue实例的选项实行合并策略。

### global-api/extend.js
```
export function initExtend (Vue: GlobalAPI) {
  Vue.cid = 0
  let cid = 1

  Vue.extend = function (extendOptions: Object): Function {
    extendOptions = extendOptions || {}
    const Super = this
    const SuperId = Super.cid
    const cachedCtors = extendOptions._Ctor || (extendOptions._Ctor = {})
    if (cachedCtors[SuperId]) {
      return cachedCtors[SuperId]
    }

    const name = extendOptions.name || Super.options.name

    const Sub = function VueComponent (options) {
      this._init(options)
    }
    Sub.prototype = Object.create(Super.prototype)
    Sub.prototype.constructor = Sub
    Sub.cid = cid++
    Sub.options = mergeOptions(
      Super.options,
      extendOptions
    )
    Sub['super'] = Super

    if (Sub.options.props) {
      initProps(Sub)
    }
    if (Sub.options.computed) {
      initComputed(Sub)
    }

    Sub.extend = Super.extend
    Sub.mixin = Super.mixin
    Sub.use = Super.use

    ASSET_TYPES.forEach(function (type) {
      Sub[type] = Super[type]
    })
    if (name) {
      Sub.options.components[name] = Sub
    }

    Sub.superOptions = Super.options
    Sub.extendOptions = extendOptions
    Sub.sealedOptions = extend({}, Sub.options)

    cachedCtors[SuperId] = Sub
    return Sub
  }
}

function initProps (Comp) {
  const props = Comp.options.props
  for (const key in props) {
    proxy(Comp.prototype, `_props`, key)
  }
}

function initComputed (Comp) {
  const computed = Comp.options.computed
  for (const key in computed) {
    defineComputed(Comp.prototype, key, computed[key])
  }
}
```
首先是给Vue一个cid为0，之后创建的子类cid会不断递增。然后是给父类和子类之间的关系绑定，这是JavaScript原型链的一些基本方法，感受到再大型的框架都是从基本的基础知识开始搭建起的~之后是为子类初始化并进行父选项和传入的子选项进行合并，我们看看子类得到了什么属性

```
Sub.cid 
Sub.options
Sub.extend
Sub.mixin
Sub.use
Sub.component
Sub.directive
Sub.filter
// 新增的
Sub.super  // 父类
Sub.superOptions // 父类选项
Sub.extendOptions  // 传入子类选项
Sub.sealedOptions // 子类目前的所有选项（父+自己）
```
### global-api/assets.js
```
export function initAssetRegisters (Vue: GlobalAPI) {
  ASSET_TYPES.forEach(type => {
    Vue[type] = function (
      id: string,
      definition: Function | Object
    ): Function | Object | void {
      if (!definition) {
        return this.options[type + 's'][id]
      } else {
        if (type === 'component' && isPlainObject(definition)) {
          definition.name = definition.name || id
          definition = this.options._base.extend(definition)
        }
        if (type === 'directive' && typeof definition === 'function') {
          definition = { bind: definition, update: definition }
        }
        this.options[type + 's'][id] = definition
        return definition
      }
    }
  })
}
```
刚提到了资源`ASSET_TYPES`指的就是组件`component`，指令`directive`，过滤器`filter`这三个东东，

#### component
这个选项就是就将传来的component作为子类继承到父类Vue实例上，大概这样用：

`Vue.component('my-component', Vue.extend({ /* ... */ }))`

#### directive
这个选项是为Vue实例添加指令，注册后指令函数被 `bind` 和 `update` 调用

#### filter
filter选项用于注册或获取全局过滤器。
```
// 注册
Vue.filter('my-filter', function (value) {
  // 返回处理后的值
})
```

关于GlobalAPI的学习就到这里啦，大大小小的东西都过了一遍，应该还蛮好理解的，有了这个更加深入的理解对我们使用全局API的时候会有很大帮助噢~如有错误之处请随时提出~谢谢~
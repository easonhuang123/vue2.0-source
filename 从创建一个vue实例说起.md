回顾上一章，我们找到了学习vue的起点——`core/instance/index`文件，我们将从创建一个vue实例开始学习~

### _init ###

在`core/instance/index`中，先看Vue函数

	function Vue (options) {
	  if (process.env.NODE_ENV !== 'production' &&
	    !(this instanceof Vue)
	  ) {
	    warn('Vue is a constructor and should be called with the `new` keyword')
	  }
	  this._init(options)
	}

里面有个`_init`函数，传入options参数，让我想起了我们在创建一个vue实例的时候都会输入el,data,props,methods啊这些参数，凭直觉我点开个同级目录的init.js找到了`_init()`这个函数，我们等会看。

接着是
	
	initMixin(Vue)
	stateMixin(Vue)
	eventsMixin(Vue)
	lifecycleMixin(Vue)
	renderMixin(Vue)

可以猜测到这些东东是对vue原型绑定一些实例方法，等一下我们一个个看。

我们去到init文件中看回`_init`函数

	// a uid 给实例一个唯一的ID
    vm._uid = uid++

    let startTag, endTag
    /* istanbul ignore if 性能指标相关的操作，不用管*/ 
    if (process.env.NODE_ENV !== 'production' && config.performance && mark) {
      startTag = `vue-perf-start:${vm._uid}`
      endTag = `vue-perf-end:${vm._uid}`
      mark(startTag)
    }

    // a flag to avoid this being observed 将_isVue设为true，避免监听对象时被监听
    vm._isVue = true


然后我们看看我们的第一道菜——对象合并策略（其实我自己也没有很懂，，还在琢磨中）

### resolveConstructorOptions ###

    if (options && options._isComponent) {
      initInternalComponent(vm, options)
    } else {
      vm.$options = mergeOptions(
        resolveConstructorOptions(vm.constructor),
        options || {},
        vm
      )
    }

由注释得知，`_isComponent = true `是指内部实例，所以`initInternalComponent()`函数是对内部实例进行优化的，我们自己写的实例应该走普通程序 `mergeOptions()`，我们又找找这个函数的来源是util/options.js文件，继续！

找到`mergeOptions()`函数后看到了前面的注释让我有点小激动

	/**
	 * Merge two option objects into a new one.
	 * Core utility used in both instantiation and inheritance.
	 * 这函数的功能是合并两个对象到一个新的对象中，而且这是用于实例化和继承相关的核心代码。
	 */

emmm。。感觉好难的样子，我们继续看吧，慢慢来。。

首先是对组件的名字进行检测是否合法

	checkComponents(child)

接着对props，inject，directive转化为对象的形式

	normalizeProps(child, vm)
	normalizeInject(child, vm)
	normalizeDirectives(child)

暂且跳过extend和mixin，接着看下面

	const options = {}
	let key
	for (key in parent) {
		mergeField(key)
	}
	for (key in child) {
		if (!hasOwn(parent, key)) {
		 	mergeField(key)
		}
	}
	function mergeField (key) {
		const strat = strats[key] || defaultStrat
		options[key] = strat(parent[key], child[key], vm, key)
	}
	return options

这里有个strats，找了一下它是来自config文件的optionMergeStrategies对象（空的）

	/**
	 * Option overwriting strategies are functions that handle
	 * how to merge a parent option value and a child option
	 * value into the final value.
	 */
	const strats = config.optionMergeStrategies

然后看完整体的文件后得知原理是把父子组件的options选项（props，data那些）进行合并再放到strats对象中，然后不同选项有对应的不同合并策略

对于data选项，是通过`mergeDataOrFn()`函数来合并

	export function mergeDataOrFn (
	  parentVal: any,
	  childVal: any,
	  vm?: Component
	): ?Function {
	  if (!vm) {
	    // in a Vue.extend merge, both should be functions
	    if (!childVal) {
	      return parentVal
	    }
	    if (!parentVal) {
	      return childVal
	    }
	    return function mergedDataFn () {
	      return mergeData(
	        typeof childVal === 'function' ? childVal.call(this) : childVal,
	        typeof parentVal === 'function' ? parentVal.call(this) : parentVal
	      )
	    }
	  } else {
	    return function mergedInstanceDataFn () {
	      // instance merge
	      const instanceData = typeof childVal === 'function'
	        ? childVal.call(vm)
	        : childVal
	      const defaultData = typeof parentVal === 'function'
	        ? parentVal.call(vm)
	        : parentVal
	      if (instanceData) {
	        return mergeData(instanceData, defaultData)
	      } else {
	        return defaultData
	      }
	    }
	  }
	}

按我的理解就是看子组件的data是不是一个函数，是的话就直接执行，然后再用`mergeData()`函数进行合并

	function mergeData (to: Object, from: ?Object): Object {
	  if (!from) return to
	  let key, toVal, fromVal
	  const keys = Object.keys(from)
	  for (let i = 0; i < keys.length; i++) {
	    key = keys[i]
	    toVal = to[key]
	    fromVal = from[key]
	    if (!hasOwn(to, key)) {
	      set(to, key, fromVal)
	    } else if (isPlainObject(toVal) && isPlainObject(fromVal)) {
	      mergeData(toVal, fromVal)
	    }
	  }
	  return to
	}


然后生命周期钩子是通过合并数组的方式

	function mergeHook (
	  parentVal: ?Array<Function>,
	  childVal: ?Function | ?Array<Function>
	): ?Array<Function> {
	  return childVal
	    ? parentVal
	      ? parentVal.concat(childVal)
	      : Array.isArray(childVal)
	        ? childVal
	        : [childVal]
	    : parentVal
	}
	
	LIFECYCLE_HOOKS.forEach(hook => {
	  strats[hook] = mergeHook
	})

而_assetTypes就是components、directives、filters这三个东东（我网上看到的，还不知道为什么，逃...)

	function mergeAssets (
	  parentVal: ?Object,
	  childVal: ?Object,
	  vm?: Component,
	  key: string
	): Object {
	  const res = Object.create(parentVal || null)
	  if (childVal) {
	    process.env.NODE_ENV !== 'production' && assertObjectType(key, childVal, vm)
	    return extend(res, childVal)
	  } else {
	    return res
	  }
	}
	
	ASSET_TYPES.forEach(function (type) {
	  strats[type + 's'] = mergeAssets
	})

`extend()`函数我翻了一下是在shared/util文件中，其实就是把一个对象的属性复制给了一个新的空对象，名副其实的extend

	export function extend (to: Object, _from: ?Object): Object {
	  for (const key in _from) {
	    to[key] = _from[key]
	  }
	  return to
	}

接下来其他属性也是类似的方法，watch是通过数组合并的方法，props，methods，inject，computed是通过extend函数进行拓展，假如父子组件有重名的，父组件的选项会被子组件的覆盖。

终于，终于看过了一个重要函数，我们回头看`_init()`，我们走到了`mergeOptions()`，传递了三个参数

    vm.$options = mergeOptions(
      resolveConstructorOptions(vm.constructor),
      options || {},
      vm
    )

我们看`resolveConstructorOptions()`

	export function resolveConstructorOptions (Ctor: Class<Component>) {
	  let options = Ctor.options
	  if (Ctor.super) {
	    const superOptions = resolveConstructorOptions(Ctor.super)
	    const cachedSuperOptions = Ctor.superOptions
	    if (superOptions !== cachedSuperOptions) {
	      Ctor.superOptions = superOptions
	      const modifiedOptions = resolveModifiedOptions(Ctor)
	      if (modifiedOptions) {
	        extend(Ctor.extendOptions, modifiedOptions)
	      }
	      options = Ctor.options = mergeOptions(superOptions, Ctor.extendOptions)
	      if (options.name) {
	        options.components[options.name] = Ctor
	      }
	    }
	  }
	  return options
	}

`vm.constructor`就是Vue对象本身，`Ctor.super`说明了Ctor是通过Vue.extend()方法创建的子类，`superOptions`递归调用了`resolveConstructorOptions`来获得父级的`options`，如果`superOptions`与`options`不同时说明父级的`options`属性值改变了，所以要更新Ctor上`options`相关属性

	if (superOptions !== cachedSuperOptions) {
		...	
	}

然后会看到`Ctor.superOptions`，`Ctor.extendOptions`，`Ctor.sealedOptions`，这些我们没见过的东西，通过全局搜索，得知它们都源于global-api/extend.js中（这个文件要找时间另说），分别是父构造函数选项，extend定义时传入的选项，目前子构造函数的选项（super+extend），这样就清晰多了，`resolveModifiedOptions`得到一个目前选项有而之前的选项sealedOptions没有的值的对象，然后进行合并得到一个准确的`vm.constructor.options`。

天啊看了这么久才弄明白_init的一小段代码，但是这段是很有价值的代码，高大上地称之为“使用策略对象合并参数选项”~嗯有个博主是这样称呼的，我表示认同。

继续吧，接下来的代码看起来会让人比较开心。

	initLifecycle(vm)
	initEvents(vm)
	initRender(vm)
	callHook(vm, 'beforeCreate')
	initInjections(vm) // resolve injections before data/props
	initState(vm)
	initProvide(vm) // resolve provide after data/props
	callHook(vm, 'created')


从这一连串的操作的执行顺序来看，先是初始化了生命周期和事件，然后调用beforeCreate钩子，再初始化inject，state和provide，然后再调用created钩子。我们一个一个进到相应的文件中看看代码

### initLifecycle ###

	export function initLifecycle (vm: Component) {
	  const options = vm.$options
	
	  let parent = options.parent
	  if (parent && !options.abstract) {
	    while (parent.$options.abstract && parent.$parent) {
	      parent = parent.$parent
	    }
	    parent.$children.push(vm)
	  }
	
	  vm.$parent = parent
	  vm.$root = parent ? parent.$root : vm
	
	  vm.$children = []
	  vm.$refs = {}
	
	  vm._watcher = null
	  vm._inactive = null
	  vm._directInactive = false
	  vm._isMounted = false
	  vm._isDestroyed = false
	  vm._isBeingDestroyed = false
	}

这个函数主要是为vue实例添加了$parent,$root,$children,$ref属性，以及一些生命周期相关的标识属性

### initEvents ###

	export function initEvents (vm: Component) {
	  vm._events = Object.create(null)
	  vm._hasHookEvent = false
	  // init parent attached events
	  const listeners = vm.$options._parentListeners
	  if (listeners) {
	    updateComponentListeners(vm, listeners)
	  }
	}

这个函数初始化一些与事件相关的属性，暂时还看不出什么东西来，先往下看


### initRender ###

	export function initRender (vm: Component) {
	  vm._vnode = null // the root of the child tree
	  const options = vm.$options
	  const parentVnode = vm.$vnode = options._parentVnode // the placeholder node in parent tree
	  const renderContext = parentVnode && parentVnode.context
	  vm.$slots = resolveSlots(options._renderChildren, renderContext)
	  vm.$scopedSlots = emptyObject
	  
	  vm._c = (a, b, c, d) => createElement(vm, a, b, c, d, false)
	  
	  vm.$createElement = (a, b, c, d) => createElement(vm, a, b, c, d, true)
	
	  const parentData = parentVnode && parentVnode.data
	
	  /* istanbul ignore else */
	  if (process.env.NODE_ENV !== 'production') {
	    defineReactive(vm, '$attrs', parentData && parentData.attrs || emptyObject, () => {
	      !isUpdatingChildComponent && warn(`$attrs is readonly.`, vm)
	    }, true)
	    defineReactive(vm, '$listeners', options._parentListeners || emptyObject, () => {
	      !isUpdatingChildComponent && warn(`$listeners is readonly.`, vm)
	    }, true)
	  } else {
	    defineReactive(vm, '$attrs', parentData && parentData.attrs || emptyObject, null, true)
	    defineReactive(vm, '$listeners', options._parentListeners || emptyObject, null, true)
	  }
	}

这个函数添加了一些$vnode，$slot，$createElement等与虚拟dom有关的方法和属性

### initInjections和initProvide ###

emmm...这个我还没有弄懂，以后再看看

### initState ###

	export function initState (vm: Component) {
	  vm._watchers = []
	  const opts = vm.$options
	  if (opts.props) initProps(vm, opts.props)
	  if (opts.methods) initMethods(vm, opts.methods)
	  if (opts.data) {
	    initData(vm)
	  } else {
	    observe(vm._data = {}, true /* asRootData */)
	  }
	  if (opts.computed) initComputed(vm, opts.computed)
	  if (opts.watch && opts.watch !== nativeWatch) {
	    initWatch(vm, opts.watch)
	  }
	}

可以看出这对props、methods、data、computed、watch等数据进行初始化，我们先分别简单地看一下

### initProps ###
```
function initProps (vm: Component, propsOptions: Object) {
  const propsData = vm.$options.propsData || {}
  const props = vm._props = {}
  const keys = vm.$options._propKeys = []
  const isRoot = !vm.$parent
  observerState.shouldConvert = isRoot
  for (const key in propsOptions) {
    keys.push(key)
    const value = validateProp(key, propsOptions, propsData, vm)
    /* istanbul ignore else */
    if (process.env.NODE_ENV !== 'production') {
      const hyphenatedKey = hyphenate(key)
      if (isReservedAttribute(hyphenatedKey) ||
          config.isReservedAttr(hyphenatedKey)) {
        warn(
          `"${hyphenatedKey}" is a reserved attribute and cannot be used as component prop.`,
          vm
        )
      }
      defineReactive(props, key, value, () => {
        if (vm.$parent && !isUpdatingChildComponent) {
          warn(
            `Avoid mutating a prop directly since the value will be ` +
            `overwritten whenever the parent component re-renders. ` +
            `Instead, use a data or computed property based on the prop's ` +
            `value. Prop being mutated: "${key}"`,
            vm
          )
        }
      })
    } else {
      defineReactive(props, key, value)
    }
    if (!(key in vm)) {
      proxy(vm, `_props`, key)
    }
  }
  observerState.shouldConvert = true
}
```
这么长的函数我抽取了一些关键的部分，`initProps`创建了变量`_props`，对传入的prop数据进行`defineReactive`操作，若是通过`Vue.extend()`得来的prop只管给它添加getter和setter就好，`defineReactive`是对数据进行双向绑定的操作，我们后面会具体学习一下。

### initMethods ###

继续看`initMethods`，这个不难理解，大概就是把这些函数绑定在vm实例上。
```
function initMethods (vm: Component, methods: Object) {
  const props = vm.$options.props
  for (const key in methods) {
    if (process.env.NODE_ENV !== 'production') {
      if (methods[key] == null) {
        warn(
          `Method "${key}" has an undefined value in the component definition. ` +
          `Did you reference the function correctly?`,
          vm
        )
      }
      if (props && hasOwn(props, key)) {
        warn(
          `Method "${key}" has already been defined as a prop.`,
          vm
        )
      }
      if ((key in vm) && isReserved(key)) {
        warn(
          `Method "${key}" conflicts with an existing Vue instance method. ` +
          `Avoid defining component methods that start with _ or $.`
        )
      }
    }
    vm[key] = methods[key] == null ? noop : bind(methods[key], vm)
  }
}
```
重点在最后一句`vm[key] = methods[key] == null ? noop : bind(methods[key], vm)`

### initData ###

接下来是data：
```
function initData (vm: Component) {
  let data = vm.$options.data
  data = vm._data = typeof data === 'function'
    ? getData(data, vm)
    : data || {}
  if (!isPlainObject(data)) {
    data = {}
    process.env.NODE_ENV !== 'production' && warn(
      'data functions should return an object:\n' +
      'https://vuejs.org/v2/guide/components.html#data-Must-Be-a-Function',
      vm
    )
  }
  // proxy data on instance
  const keys = Object.keys(data)
  const props = vm.$options.props
  const methods = vm.$options.methods
  let i = keys.length
  while (i--) {
    const key = keys[i]
    if (process.env.NODE_ENV !== 'production') {
      if (methods && hasOwn(methods, key)) {
        warn(
          `Method "${key}" has already been defined as a data property.`,
          vm
        )
      }
    }
    if (props && hasOwn(props, key)) {
      process.env.NODE_ENV !== 'production' && warn(
        `The data property "${key}" is already declared as a prop. ` +
        `Use prop default value instead.`,
        vm
      )
    } else if (!isReserved(key)) {
      proxy(vm, `_data`, key)
    }
  }
  // observe data
  observe(data, true /* asRootData */)
}
```

先是对data的类型进行检查是否是function，是的话就转化为对象data，然后将其赋给变量_data，然后对data中变量名已经校验后，执行`observe(data, true /* asRootData */)`,`observe`是一个负责对数据进行监听然后触发watcher的东东，我们也把它放到后面数据双向绑定的内容中再进行分析。

### initComputed && initWatch ###

然后是`initComputed`，大概就是给每个computed的值创建一个watcher，然后给它们添加setter和getter。

最后一个`initWatch`为每个watch中的变量执行`vm.$watch()`
```
Vue.prototype.$watch = function (
  expOrFn: string | Function,
  cb: any,
  options?: Object
): Function {
  const vm: Component = this
  if (isPlainObject(cb)) {
    return createWatcher(vm, expOrFn, cb, options)
  }
  options = options || {}
  options.user = true
  const watcher = new Watcher(vm, expOrFn, cb, options)
  if (options.immediate) {
    cb.call(vm, watcher.value)
  }
  return function unwatchFn () {
    watcher.teardown()
  }
}
```

`vm.$watch()`也是创建了watcher实例，创建完成后再将其销毁。

`initState`的流程基本过了一遍，对数据初始化并进行双向绑定，然后这些都初始化完成后调用了created生命钩子`callHook(vm, 'created')`，可见在created阶段vue做了这一系列的初始化以及数据的绑定。

### $mount

回到`_init`函数中，在初始化完成调用created钩子之后接下来的就是
```
if (vm.$options.el) {
  vm.$mount(vm.$options.el)
}
```
若传入的选项中有指定的dom节点，那么我们执行`$mount`函数，也就是说将实例绑定在指定的节点中。
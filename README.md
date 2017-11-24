## 目录结构 ##

这是目前我自己看过源码的一些目录（不完整）

- build：
	- alias 定义别名
	- config 定义各种构建方式
	- build
	
- dist
	- vue.js 构建后生成的文件，npm run dev后生成vue.js用于调试
	
- flow
	- flow是Facebook出品的，针对JavaScript的静态类型检查工具,为Javascript增加强类型系统的支持
- src:
	- compiler 编译器 将模板字符串转化为AST，由AST再转化为本地代码，其中对代码进行优化
	- core
		- components
		- global-api
		- instance 实例初始化
		- observer 数据观察，响应式，双向绑定
		- util 工具类
		- vdom 虚拟dom
	- platforms 各种使用vue的平台
	- server 服务端使用



## 从入口文件开始学习vue ##

学习vue源码，把源码下下来之后第一件事当然是把项目跑起来啦，看看什么回事

	npm install
	npm run dev 

我在运行第二条语句的时候出现了错误
    
    Error: Could not load C:\vue\src\core/config (imported by C:\vue\src\platforms\web\entry-runtime-with-compiler.js): ENOENT: no such file
    or directory, open 'C:\vue\src\core\config'

然后发现是rollup-plugin-alias插件中路径'/' 应该改为'\'，估计是有人提PR出现bug了，解决方法是将` node_modules/rollup-plugin-alias/dist/rollup-plugin-alias.js` 第81行：
`var entry = options[toReplace];` 改为 `var entry = normalizeId(options[toReplace]);` 就没问题啦~

run成功后会在`dist`文件夹中生成`vue.js`文件，有了这个文件，我们就可以通过引入它来写一下demo，边写栗子边打断点边看源码，这样会更加便于我们对源码的了解哦~

然后我们看一下`package.json`文件`script`里面`dev`指令究竟做了什么：

    rollup -w -c build/config.js --environment TARGET:web-full-dev

这时候又有疑惑了，`rollup`又是啥，然后又谷歌了一番，得知这玩意儿大概也是一个前端模块的打包利器，通过tree-shaking的方式让你的bundle最小化（这是多亏es6中解构赋值的语法才形成的好方法），它还提供了sourcemap功能（其实我感觉和webpack差不多，就webpack实现了代码拆分按需加载，而rollup生成更简洁的代码一次性执行）

再看回上文的指令，`-w` 指的是watch，`-c build/config.js` 指定配置文件是build/config.js，`--environment TARGET:web-full-dev`指的是process.env.TARGET的值等于 ‘web-full-dev’，emmm..接下来我们进到build/config.js看看里面是啥

我找到了`web-full-dev`的对应代码

	// Runtime+compiler development build (Browser)
	  'web-full-dev': {
	    entry: resolve('web/entry-runtime-with-compiler.js'),
	    dest: resolve('dist/vue.js'),
	    format: 'umd',
	    env: 'development',
	    alias: { he: './entity-decoder' },
	    banner
	  },


所以跟着代码走到了`genConfig`函数，得知这个文件的输出结果是

	module.exports = genConfig({
	    entry: path.resolve(__dirname, '../src/platforms/web/web-runtime-with-compiler.js'),
	    dest: path.resolve(__dirname, '../dist/vue.js'),
	    format: 'umd',
	    env: 'development',
	    alias: { he: './entity-decoder' },
	    banner
	})

所以入口文件就是`../src/platforms/web/web-runtime-with-compiler.js`，现在我们就可以一步一步寻找vue的足迹啦。

从入口文件开始寻找它是从哪里import出vue的，然后一步一步的找

	import Vue from './runtime/index'
到

	import Vue from 'core/index'
再到

	import Vue from './instance/index'

终于...找到了core/instance/index文件，有种找到家的感觉有木有！instance文件夹从名字可以猜出应该是与vue实例初始化相关的文件，我们就从这里开始我们的vue之旅咯~！

### _init ###

整体看一眼index文件，先看Vue函数

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

	// merge options
    if (options && options._isComponent) {
      // optimize internal component instantiation
      // since dynamic options merging is pretty slow, and none of the
      // internal component options needs special treatment.
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

哭哭，感觉好难的样子，我们继续看吧，慢慢来。。

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

	/**
	 * Data
	 */
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
	    // when parentVal & childVal are both present,
	    // we need to return a function that returns the
	    // merged result of both functions... no need to
	    // check if parentVal is a function here because
	    // it has to be a function to pass previous merges.
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

	/**
	 * Helper that recursively merges two data objects together.
	 */
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

	/**
	 * Hooks and props are merged as arrays.
	 */
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

	/**
	 * Assets
	 *
	 * When a vm is present (instance creation), we need to do
	 * a three-way merge between constructor options, instance
	 * options and parent options.
	 */
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

	/**
	 * Mix properties into target object.
	 */
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
	      // super option changed,
	      // need to resolve new options.
	      Ctor.superOptions = superOptions
	      // check if there are any late-modified/attached options (#4976)
	      const modifiedOptions = resolveModifiedOptions(Ctor)
	      // update base extend options
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
	
	  // locate first non-abstract parent
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
	  // bind the createElement fn to this instance
	  // so that we get proper render context inside it.
	  // args order: tag, data, children, normalizationType, alwaysNormalize
	  // internal version is used by render functions compiled from templates
	  vm._c = (a, b, c, d) => createElement(vm, a, b, c, d, false)
	  // normalization is always applied for the public version, used in
	  // user-written render functions.
	  vm.$createElement = (a, b, c, d) => createElement(vm, a, b, c, d, true)
	
	  // $attrs & $listeners are exposed for easier HOC creation.
	  // they need to be reactive so that HOCs using them are always updated
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

这个就厉害了，对props、methods、data、computed、watch等（起码我看得懂的）数据进行初始化，我们下次可以重点讨论一下这个~




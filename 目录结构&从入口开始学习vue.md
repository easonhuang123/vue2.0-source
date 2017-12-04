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

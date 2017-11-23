# vue2.0源码学习 #

## 目录结构 ##


- build：
	- alias 定义别名
	- config 定义各种构建方式
	- build
	
- dist
	- vue.js 构建后生成的文件，npm run dev后生成vue.js用于调试
	
- flow
	- flow是Facebook出品的，针对JavaScript的静态类型检查工具,为Javascript增加强类型系统的支持
- src:
	- compiler
	- core
		- components
		- global-api
		- instance 实例初始化
		- observer 数据观察，响应式，双向绑定
		- util 工具类
		- vdom 虚拟dom
	- platforms 各种使用vue的平台
	- server 服务端使用




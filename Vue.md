# 1. 响应式原理

## Observer

观察者，使用`Object.defineProperty`方法对对象的每一个子属性进行数据劫持/监听，在`get`方法中进行依赖收集，添加订阅者`watcher`到订阅中心。在`set`方法中，对新的值进行收集，同时订阅中心通知订阅者们。

```js
// 对象的子对象递归进行Observe并返回子节点的Observer对象
let childOb = observe(val)
Object.defineProperty(obj, key, {
  enumerable: true,
  configurable: true,
  get: function reactiveGetter () {
    // 如果原本对象拥有getter方法则执行
    const value = getter ? getter.call(obj) : val
    if (Dep.target) {
      // 进行依赖收集
      dep.depend()
      if (childOb) {
        // 子对象进行依赖收集，其实就是将同一个
      }
    }
  }
})
```

在`setter`中通知观察者更新，在`getter`中向`Dep`(调度中心)添加观察者。

## Watcher

扮演的角色是订阅者，它的主要作用是为观察属性提供通知函数，当被观察的值发生变化时，会接收到来自订阅中心`dep`的通知，从而触发依赖更新。

核心方法有：`get()`获得`getter`的值并且重新进行依赖收集`addDep(dep: Dep)`添加一个依赖关系到订阅中心`Dep`集合中`update()`提供给订阅中心的通知接口，如果不是同步的`(sync)`，那么会放到队列中，异步执行，在下一个事件循环中执行（采用`Promise, MutationObserver, setTimeout`来异步执行）

## Dep

扮演的角色是调度中心，主要的作用就是收集观察者`Watcher`和通知观察者目标更新。每一个属性都有一个`Dep`对象，用于存放所有订阅了该属性的观察者对象，当数据发生改变时，会遍历观察者列表`dep.subs`，通知所有的`watcher`，让订阅者执行自己的`update`逻辑。


# 2. computed 为什么比 watch method 性能要好

从编码上 `computed` 实现的功能也可以通过普通`method`实现，但与函数相比，计算属性是基于响应式依赖进行缓存的，只有在依赖的数据发生改变时，才重新进行计算，只要依赖没有发生变化，多次访问都只是从缓存中获取。

计算属性是基于`watcher`实现，看看源码

```js
// 初始化 computed
// 核心是为每个计算属性创建一个 watcher 对象
function initComputed (vm: Component, computed: Object) {
  const watchers = vm._computedWatchers = Object.create(null)

  for (const key in computed) {
    const userDef = computed[key]

    // 计算属性可能是一个function，也有可能设置了get以及set的对象
    let getter = typeof userDef === 'function' ? userDef : userDef.get
    if (process.env.NODE_ENV !== 'production') {
      // getter不存在的时候抛出warning并且给getter赋空函数
      if (getter === undefined) {
        warn(`No getter function has been defined for computed property ${key}`, vm)
        getter = noop
      }
    }
  }
}
```

`computed` 和 `watch` 主要区别在于使用场景，计算属性更适合于模板渲染，依赖其他对象值的变化，做重新计算再渲染，监听多个值来改变一个值。而监听属性`watch`，用于监听某一个值的变化，进行一系列复杂的操作。监听属性可以支持异步，计算属性只能是同步。


# 3. Vue对数组的处理

在官方文档上关于数组的注意事项有这么一段

> 由于 JavaScript 的限制，Vue不能检测以下数组的变动：
1. 当你利用索引直接设置一个数组项时，例如：`vm.item[indexOfItem] = newValue`
2. 当你修改数组的长度时，例如：`vm.item.length = newLength`

先看第二点，这是因为`Object.defineProperty`不能监听数组的长度，所以直接修改数组长度是没法被监听到的。

关于第一点，我们看看源码的实现

```js
// vue\src\core\observer\index.js

constructor(value: any) {
  this.value = value
  this.dep = new Dep()
  this.vmCount = 0
  def(value, '__ob__', this)
  // 在Observer构造函数中，对数组类型进行特殊处理
  if (Array.isArray(value)) {
    if (hasProto) {

    }
  }
}
```

`observeArray`是对数组中的值进行监听，并不是数组下标，所以通过索引来修改值是监听不到的，假如是通过监听索引的话，那是可以实现的。那为什么不监听下标呢？据说是出于性能考虑。

因为是对数组元素做的监听，那么数组api造成的修改自然就没法监听到了，所以vue对数组的方法进行了变异，包裹了一层，本质还是执行数组的api

```js
// vue\src\core\observer\array.js
const methodsToPatch = [
  'push', 'pop', 'shift', 'unshift', 'splice', 'reverse', 'sort'
]

```

# 4. Vue中 key 的作用

有两点用处：快速节点比对和节点唯一标识

## 利用快速节点对比

用作于 `vnode` 的唯一标识，便于更快更准确的在旧节点列表中查找节点

在内部对两节点进行比较的时候，会有限判断key是否一致，如下，如果Key不一致，立马得到结果

## 列表节点唯一标识

列表循环 `v-for="i in dataList"`会有提示我们需要加上`key`，因为循环后的`dom`节点的结构没特殊处理的话是相同的，key的默认值是`undefined`，那么按照上面`sameVnode`的算法，新生成的Vnode与旧节点的比较结果就是相同的，vue会对这些节点就地修改/复用相同类型元素的，这种模式是高效的，但是这种模式会有副作用，比如节点是带有状态的，那么就会出现异常的bug，所以这种不写`key`的默认处理只适用于不依赖其他状态的列表。

## vuex中为什么把异步操作封装在action，把同步操作放在mutations?

`mutations`同步主要是为了能用`devtools`跟踪状态的变化，每次执行完后，就可以立即得到下一个状态，这样在`devtools`调试工具中，就可以跟踪到状态的每一次变化，可以做时间旅行`time-travel`，那么如果是异步的话，就没法知道状态什么时候被更新，所以就有了一个`actions`用来专门处理异步函数，但要求状态的需要触发`mutations`，这样一来对于异步的更新也可以清晰看到状态的流转。
参考文档：[https://www.zhihu.com/question/48759748/answer/112823337](https://www.zhihu.com/question/48759748/answer/112823337)


# 5. vuex getter 方法跟直接 state 中获取有什么区别

`getter` 类似于计算属性，带有缓存；当只有响应的属性发生变化才会更新缓存，相比直接获取效率更好，在设计上可以便于抽象逻辑。


# 6. vue-router中的link跳转和a链接跳转的区别
- 判断是否有 `onclick` 事件，有就执行
- 阻止默认事件
- 使用 `history.replace` 或 `history.push` 修改地址栏，同时不会触发页面刷新

# 7. webpack 中的Loader和Plugin是干什么的
loader: 对文件进行转换，比如说将ts编译成js，css预处理等等plugin: 通过监听webpack运行中的广播事件，从而进行自己的操作，如常用的`HtmlWebpackPlugin`: 创建html文件、`webpack.optimize.UglifyJsPlugin`: 混淆压缩。

# 8. webpack的externals

防止将外部引用的包打包到bundle中，而是在运行时通过模块化的方式从外部引用。比如我们通过cdn引用jquery，我们不希望jq打包到bundle中，而且在使用时希望能通过模块化的方式引用，那么可以如下配置

```js
module.exports = {
  // ...
  externals: {
    jquery: 'jQuery'
  }
}
```


# 8. 什么是前端工程化？

关于工程化，每个人都有自己的理解，以下是我个人的理解，每个点都可以展开很多点来说，这题更多对于工作中的总结。

A. 协助上
  - 统一开发规范，代码，命名规范，引用语法检测工具
  - 版本管理，提交规范
B. 项目架构上
  - 模块化、组件化、沉淀组件库，降低编码间的耦合
  - 团队统一脚手架
C. 构建
  - 资源压缩、混淆
  - 图片处理
D. 持续集成、部署
  - 减少认为参与，Jenkins CI
  - 静态资源分离，cdn，静态文件服务器
E. 质量跟踪
  - 持续的单元测试，mocha Istanbul
  - 监控
F. 用户体验
  - 性能优化，客户端，服务端，代理服务器
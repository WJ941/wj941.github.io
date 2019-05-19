---
title: vue-construct-init-mount
date: 2019-05-19 10:34:36
tags:
---

#  Vue构造函数
`vue version: 2.6.10`



开发时使用的`Vue`来源在`src/platforms`下，都以`entry`开头，本文以`src/platforms/web/entry-runtime-with-compiler.js`为例。
最后被开发者使用的`Vue`是在原始`Vue`构造函数的原型上挂载一系列方法和属性后导出的。
从`entry-runtime-with-compiler.js`里不断查看`Vue`的导入(import)，可以发现`Vue`构造函数文件路径如下：
1. ___`entry-runtime-with-compiler.js`___ 
2. ___`src/platforms/web/runtime/index`___
3. ___`src/core/index`___
4. ___`src/core/instance/index`___
   
从程序执行路径来看，`Vue`构造函数依次经历了一下过程：

## `src/core/instance/index`

### 原始构造函数
  ```
  function Vue (options) {
    if (process.env.NODE_ENV !== 'production' &&
      !(this instanceof Vue)
    ) {
      warn('Vue is a constructor and should be called with the `new` keyword')
    }
    this._init(options)
  }
  ```
  Vue构造函数来源于[`src/core/instance/index.js`](https://github.com/vuejs/vue/blob/dev/src/core/instance/index.js)，构造函数的内容很简单：
    1. 检查是否用`new`操作构造函数
    2. 调用`Vue`实例的`_init`函数

### 5个`mixin`函数
在构造函数之后，紧接着的是5个`mixin`函数
```
initMixin(Vue)
stateMixin(Vue)
eventsMixin(Vue)
lifecycleMixin(Vue)
renderMixin(Vue)
```
1. 执行`initMixin()`挂载的方法: 
  - `Vue.prototype._init`
2. 执行`stateMixin()`挂载的方法:
  - `Vue.prototype.$data`
  - `Vue.prototype.$props`
  - `Vue.prototype.$set`
  - `Vue.prototype.$delete`
  - `Vue.prototype.$watch`
3. 执行`eventsMixin()`挂载的方法:
  - `Vue.prototype.$once`
  - `Vue.prototype.$on`
  - `Vue.prototype.$off`
  - `Vue.prototype.$emit`
4. 执行`lifecycleMixin()`挂载的方法:
  - `Vue.prototype._update`
  - `Vue.prototype.$forceUpdate`
  - `Vue.prototype.$destroy`
5. 执行`renderMixin()`挂载的方法:
  - `Vue.prototype.$nextTick`
  - `Vue.prototype._render`


## `src/core/index.js`
先调用`initGlobalAPI(Vue)`，此函数挂载的方法有
```
Vue.config={}
Vue.util = {
  warn,
  extend,
  mergeOptions,
  defineReactive
}

Vue.set = set
Vue.delete = del
Vue.nextTick = nextTick
Vue.observable = () => {}
Vue.options = {
  components: { keepAlive },
  directives: {},
  filters: {}
}
Vue.use = () => {}
Vue.mixin = () => {}
Vue.extend = () => {}
Vue.component = () => {}
Vue.directive = () => {}
Vue.filter = () => {}
```
然后挂载2个原型上的方法和1个静态方法
```
Vue.prototype.$isServer
Vue.prototype.$ssrContext
Vue.FunctionalRenderContext
```
  
## `src/platforms/web/runtime/index.js`
在`src/platforms/web/runtime/index.js`里挂载的方法
```
  Vue.prototype.$mount
  Vue.options.directives = { model, show }
  Vue.options.components = { keepAlive, Transition, TransitionGroup }
  Vue.prototype.__patch__ = inBrowser ? patch : noop
```
## entry-runtime-with-compiler.js
在上一步的`$mount`基础之上加上额外功能封装`$mount`，还挂载了1个静态方法`Vue.compile`
```
  Vue.prototype.$mount
  Vue.compile
```

至此`Vue`构造函数构建完毕。

---

# new Vue()实例化过程
`Vue`的构造函数就执行一个操作`this._init()`。
## `Vue.prototype._init`函数
```
Vue.prototype._init = function (options?: Object) {
  const vm: Component = this
  vm._uid = uid++

  let startTag, endTag
  /* istanbul ignore if */
  if (process.env.NODE_ENV !== 'production' && config.performance && mark) {
    startTag = `vue-perf-start:${vm._uid}`
    endTag = `vue-perf-end:${vm._uid}`
    mark(startTag)
  }

  // a flag to avoid this being observed
  vm._isVue = true
  // merge options
  if (options && options._isComponent) {
    // optimize internal component instantiation
    // since dynamic options merging is pretty slow, and none of the
    // internal component options needs special treatment.
    initInternalComponent(vm, options)
  } else {
    console.log('resolveConstructorOptions: ', resolveConstructorOptions(vm.constructor));
    vm.$options = mergeOptions(
      resolveConstructorOptions(vm.constructor),
      options || {},
      vm
    )
  }
  /* istanbul ignore else */
  if (process.env.NODE_ENV !== 'production') {
    initProxy(vm)
  } else {
    vm._renderProxy = vm
  }
  // expose real self
  vm._self = vm
  initLifecycle(vm)
  initEvents(vm)
  initRender(vm)
  callHook(vm, 'beforeCreate')
  initInjections(vm) // resolve injections before data/props
  initState(vm)
  initProvide(vm) // resolve provide after data/props
  callHook(vm, 'created')

  /* istanbul ignore if */
  if (process.env.NODE_ENV !== 'production' && config.performance && mark) {
    vm._name = formatComponentName(vm, false)
    mark(endTag)
    measure(`vue ${vm._name} init`, startTag, endTag)
  }

  if (vm.$options.el) {
    vm.$mount(vm.$options.el)
  }
}
```
### `vm._uid = uid++`
每一个`Vue`实例都有一个id
### `vm._isVue = true`
`_isVue`在`observer`函数里被用到，如果一个值的`_isVue`为`true`，只就不会被观测
### merge options
把`Vue.options`合并到`vm.options`，每一个`Vue`实例上都会有`Vue.options`的内容。此处`Vue.options`为
```
Vue.options = {
  components: { keepAlive, Transition, TransitionGroup },
  directives: { model, show },
  filters: {},
  _base: Vue
}
```
  1. `keepAlive`在调用`src/core/index.js`的`initGlobalAPI`函数时被定义在`Vue.options`上
  2. `Transition`和`TransitionGroup`在调用`src/platforms/web/runtime/index.js`时被定义到`Vue.options`上

###  `_renderProxy`
  `_renderProxy`的作用是在render期间代替`vm`，检测引用错误。

### `vm._self=vm`
### `initLifecycle(vm)`
`initLifecycle`的作用是定义了一些实例变量
```
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
```
带`$`符号的属性都是[官方文档定义的api](https://vuejs.org/v2/api/#vm-root)

`_watcher`是在`Watcher`的构造函数中被定义，`vm`对`Watcher`实例的引用

### `initEvents(vm)`
`initEvents`定义了与事件处理有关的变量
```
vm._events = Object.create(null)
vm._hasHookEvent = false
```
### `initRender(vm)`
```
vm._vnode = null // the root of the child tree
vm._staticTrees = null // v-once cached trees
vm.$slots = {...}
vm.$scopedSlots = emptyObject
vm._c = () => {}
vm.$createElement = () => {}
vm.$attrs = {...}
vm.$listeners = {...}
```
此处`$attrs`和`$listeners`是reactive的。
### `callHook(vm, 'beforeCreate')`
### `initInjections(vm)`
把`inject`包括的属性响应式定义到实例上
### `initState`，
初始化 `props`, `methods`, `data`, `computed`和`watch`。
### `initProvide(vm)`
把`provide`属性赋值到`vm._provided`。
### `callHook(vm, 'created')`
### `$mount`


# `mount`的过程
## `entry-runtime-with-compiler.js`
这里是把`el`或者`template`里的字符串转换成`render function`并赋值到`$options.render`。
## `mountComponent`函数
1. 检查`$options.render`是否存在
2. `callHook(vm, 'beforeMount')`
3. 更新组件的函数
  ```
    updateComponent = () => {
      vm._update(vm._render(), hydrating)
    }
  ```
  `vm.render()`的作用是执行`$options.render`函数，返回一个`VNode`实例
  `vm._update`的作用是执行`vm.__patch__`函数，把`vnode`更新到`html`中
4. 生成一个新的Watcher实例
  ```
    new Watcher(vm, updateComponent, noop, {
      before () {
        if (vm._isMounted && !vm._isDestroyed) {
          callHook(vm, 'beforeUpdate')
        }
      }
    }, true /* isRenderWatcher */)
  ```

## Watcher类
### Watcher类构造函数
#### 参数
```
  constructor (
    vm: Component,
    expOrFn: string | Function
    cb: Function, 
    options?: ?Object,
    isRenderWatcher?: boolean
  ) {...}
```
1. `vm`是`Vue`实例，使用在`expOrFn`和`cb`需要用到；
2. `expOfFn`可以是一个字符串路径或者一个函数（[官方文档](https://cn.vuejs.org/v2/api/#vm-watch)）
`cb`是以`newVal`和`oldVal`为参数的回调函数。

#### 内容
构造函数里重要的是赋值`this.getter`
```
    if (typeof expOrFn === 'function') {
      this.getter = expOrFn
    } else {
      this.getter = parsePath(expOrFn)
      if (!this.getter) {
        this.getter = noop
        process.env.NODE_ENV !== 'production' && warn(...)
      }
    }
```
如果`expOrFn`是的函数，直接赋值给`getter`，如果是`string`则通过`parsePath`转换。`string`类型的`expOrFn`官方给的例子是`a.b.c`。

`parsePath`函数在`src/core/util/lang.js`
```
export function parsePath (path: string): any {
  if (bailRE.test(path)) {
    return
  }
  const segments = path.split('.')
  return function (obj) {
    for (let i = 0; i < segments.length; i++) {
      if (!obj) return
      obj = obj[segments[i]]
    }
    return obj
  }
}
```
第一步是做`unicode`检查

返回一个匿名函数赋值给`getter`，这个函数的参数是一个`obj`，`getter`在`Watcher`类的`get`方法里被调用的时候被传递的唯一参数是`vm`，相当于`vm[a][b][c]`。

再回到`Watcher`类的构造函数
```
  this.value = this.lazy
    ? undefined
    : this.get()
```
`this.lazy`是在构造函数的参数`options`定义的，如果是lazy的，只有当water被访问的时候才执行`this.get()`，否则`Watcher`实例化的时候就执行`this.get()`

#### get函数
```
  get () {
    pushTarget(this)
    let value
    const vm = this.vm
    try {
      value = this.getter.call(vm, vm)
    } catch (e) {
      if (this.user) {
        handleError(e, vm, `getter for watcher "${this.expression}"`)
      } else {
        throw e
      }
    } finally {
      // "touch" every property so they are all tracked as
      // dependencies for deep watching
      if (this.deep) {
        traverse(value)
      }
      popTarget()
      this.cleanupDeps()
    }
    return value
  }
```
`get`函数第一步就是`pushTarget(this)`，作用就是把`this`赋值到`Dep.target`,`Dep.target`是用来保存一个函数，当有响应式数据(被`defineReactive`调用过的数据)被取值（调用其`get`函数）时，会调用数据内部`dep`实例的`depend`方法收集依赖，把`Dep.target`添加到`dep.subs[]`数组里。
当响应式数据被改变时会调用数据内部`dep`实例的`notify`方法，`notify`方法行为就是遍历`dep.subs[]`数组，调用数组里每个元素（即`Watcher`实例）的`update`方法，`update`方法重新调用`watcher.getter()`完成数据更新。

对于这个过程[官方文档](https://vuejs.org/v2/guide/reactivity.html#How-Changes-Are-Tracked)给出了一个流程图
![流程图!](reactive-in-depth.png "reactive-in-depth")
在`vm._init`的过程中调用`mountComponent`，`mountComponent`的过程中实例化一个`Watcher`，就会调用`updateComponent`函数，在调用`updateComponent`的过程中就会对数据取值，比如说渲染`<h1>{{ msg }}</h1>`的过程中就会对`msg`取值，取值的过程中就会调用值的`get`方法即图中的 ___Touch___ 过程，`get`方法回调用`dep.depend`,即图中的 ___Collect as Dependency___ 过程.

当数据改变时会调用数据值的`set`方法（setter），`setter`会调用`dep.notify`，`notify`会执行之前被收集的函数，对于`Vue`实例就是`updateComponent`方法，依次执行`vm._render`和`vm._update`，完成更新。即图中的 ___Trigger re-reander___ 。
---
title: Vue源码阅读
date: 2018-08-20 13:06:39
tags:
---

Vue构造函数的options属性
```
Vue.options = {
    directives: {
        model, // src/platforms/web/runtime/directives/model.js
        show  // src/platforms/web/runtime/directives/show.js
    },
    components: {
        keepAlive, // src/core/components/keep-alive.js
        Transition, // src/platforms/web/runtime/components/transition.js
        TransitionGroup // src/platforms/web/runtime/components/transition-group.js
    },
    filters: Object.create(null),
    _base:  Vue
}
```
`mergeOptions`在`Vue.prototype._init`, `Vue.extend`和`Vue.mixin`三个函数中使用
`Vue.prototype._init`在`Vue`构造函数和`Vue.extend`中使用

# Vue的合并策略：
TL;DR
默认合并策略: 子元素没有，返回父元素的；子元素有，返回子元素的值。

在非生产模式下，`el`和`props`采取默认合并策略；

`data`：返回一个函数，函数运行返回一个对象，把父元素`data`上有但子元素`data`没有的值赋值到子元素`data`上，返回的对象就是新的子元素`data`；

生命周期钩子：相当于子元素钩子数组和父元素的钩子数组取并集；

`components`, `directives`和`filters`： 资源的合并是返回一个新对象，把父元素上的值放在新对象的原型链上，子元素的值直接赋值给新对象；

`watch`: 相当于取并集;

`props`, `methods`, `inject`和`computed`: 返回一个新对象，先把父元素的值赋值到新对象上，再遍历子元素，把子元素的值覆盖到新对象上；

`provide`： 和`data`类似。

## 合并`el`和`props`：
```
if (process.env.NODE_ENV !== 'production') {
  strats.el = strats.propsData = function (parent, child, vm, key) {
    if (!vm) {
      warn(
        `option "${key}" can only be used during instance ` +
        'creation with the `new` keyword.'
      )
    }
    return defaultStrat(parent, child)
  }
}
```
`el`和`props`的合并只有在`new Vue()`时使用，策略使用默认策略：子元素没有，返回父元素的；子元素有，返回子元素的值。
## 合并`data`:
```
strats.data = function (
  parentVal: any,
  childVal: any,
  vm?: Component
): ?Function {
  if (!vm) {
    if (childVal && typeof childVal !== 'function') {
      process.env.NODE_ENV !== 'production' && warn(
        'The "data" option should be a function ' +
        'that returns a per-instance value in component ' +
        'definitions.',
        vm
      )
      
      return parentVal
    }
    return mergeDataOrFn(parentVal, childVal)
  }

  return mergeDataOrFn(parentVal, childVal, vm)
}
```
`if (!vm) `和`if (childVal && typeof childVal !== 'function')`表明在`Vue.extend`和`Vue.mixin`中`data`必须
为一个函数（Vue构造函数中没有data属性，所以只检查childVal）。
```
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
        typeof childVal === 'function' ? childVal.call(this, this) : childVal,
        typeof parentVal === 'function' ? parentVal.call(this, this) : parentVal
      )
    }
  } else {
    return function mergedInstanceDataFn () {
      // instance merge
      const instanceData = typeof childVal === 'function'
        ? childVal.call(vm, vm)
        : childVal
      const defaultData = typeof parentVal === 'function'
        ? parentVal.call(vm, vm)
        : parentVal
      if (instanceData) {
        return mergeData(instanceData, defaultData)
      } else {
        return defaultData
      }
    }
  }
}
```
### 在没有`vm`实例情况下
`child.options.data`不存在返回`parent.options.data`，比如`Vue.extend({})`，`Vue.options.data`也不存在，返回一个`undefined`；
`parent.options.data`不存在但是`child.options.data`存在返回`child.options.data`，比如
```
const sub = Vue.extend({
  data: () => {
    msg: "hello world"
  }
})
```
在上一步中已经检测childVal必须是一个函数，所以返回结果是一个函数；
`parent.options.data`和`child.options.data`都存在：
```
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
```
举个例子：
```
var Parent = Vue.extend({
  data: function () {
    return {
      person: { firstName: 'parent first name', lastName: 'parent last name', },
      parentHave: "parent have",
    }
  }
})
var Child = Parent.extend({
  data: function () {
    return {
      person: { firstName: 'child first name',  },
      chidlHave: "child have",
    }
  }
})
```
合并后`Child.options`应该等于
```
function mergedDataFn {
  return {
    person: { firstName: 'child first name', lastName: 'parent last name',},
    chidlHave: "child have",
    parentHave: "parent have",
  }
}
```
在devtools中的结果：
{% asset_img merge_data_result_devtools.png %}

### 在有`vm`实例的情况下
返回一个`mergedInstanceDataFn`函数，返回调用`mergeData`函数合并`new Constructor(options)`时提供的`options.data`和`Constructor.options.data`的值。

```
var Parent = {
  data: function() {
    return {
      parent: "parent",
      both  : "both parent"
    }
  }
}
var Child = new Vue({
  mixins: [Parent],
  data: function() {
    return {
      child: "child",
      both  : "child parent"
    }
  }
})
```
总之`data`合并是返回一个函数，运行这个函数返回一个对象，把父元素`data`上有但子元素`data`没有的值赋值到子元素`data`上，返回对象等于新的子元素的`data`。
看下结果

{% asset_img merge_instance_data_result_devtools.png %}

## 合并生命周期钩子函数

```
export const LIFECYCLE_HOOKS = [
  'beforeCreate',
  'created',
  'beforeMount',
  'mounted',
  'beforeUpdate',
  'updated',
  'beforeDestroy',
  'destroyed',
  'activated',
  'deactivated',
  'errorCaptured'
]

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
```
如果`childVal`不存在，返回`parentVal`,
如果`childVal`存在但是`parentVal`不存在，返回`[childVal]`,
如果`childVal`和`parentVal`都存在，返回`parentVal.concat(childVal)`
举个例子
```
var Parent = Vue.extend({
  mounted: function parentMounted() { },
  updated: function parentUpdated() { },
})
var Child = Parent.extend({
  beforeUpdate: function childBeforeUpdate() { },
  updated:      function childUpdated() { },
})
```
在运行后Child.options中钩子函数应该是
```
mounted:      [ parentMounted ],
updated:      [ parentUpdated, childUpdated],
beforeUpdate: [ childBeforeUpdate ],
```
总之钩子函数是子元素和父元素如果都有，则通过`parentVal.concat(childVal)`返回一个新数组后给子元素
在devtools中查看结果：

{% asset_img merge_hook_result_devtools.png %}

## 合并`components`, `directives`和`filters`

```
export const ASSET_TYPES = [
  'component',
  'directive',
  'filter'
]

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

export function extend (to: Object, _from: ?Object): Object {
  for (const key in _from) {
    to[key] = _from[key]
  }
  return to
}
```
总结就是返回一个新对象，把Parent的值放在原型链上，再把Child的值重新赋值到这个新对象上，最后返回这个新对象
举个例子：
```
var Parent = Vue.extend({
  components: {
    'parent-component': {},
  },
})
var Child = Parent.extend({
  components: {
    'child-component': {},
  },
})
```
总之资源的合并是返回一个新对象，把父元素上的值放在新对象的原型链上，子元素的值直接赋值给新对象。
看下结果
{% asset_img merge_components_result_devtools.png %}
（`Vue.options.components`有3个内置组件）

## 合并`watch`
```
strats.watch = function (
  parentVal: ?Object,
  childVal: ?Object,
  vm?: Component,
  key: string
): ?Object {
  // work around Firefox's Object.prototype.watch...
  if (parentVal === nativeWatch) parentVal = undefined
  if (childVal === nativeWatch) childVal = undefined
  /* istanbul ignore if */
  if (!childVal) return Object.create(parentVal || null)
  if (process.env.NODE_ENV !== 'production') {
    assertObjectType(key, childVal, vm)
  }
  if (!parentVal) return childVal
  const ret = {}
  extend(ret, parentVal)
  for (const key in childVal) {
    let parent = ret[key]
    const child = childVal[key]
    if (parent && !Array.isArray(parent)) {
      parent = [parent]
    }
    ret[key] = parent
      ? parent.concat(child)
      : Array.isArray(child) ? child : [child]
  }
  return ret
}
```
前两行`nativeWatch`是因为版本号58之前的Firefox有`Object.prototype.watch`, 为了覆盖它把`parentVal`和`parentVal`重置为`undefined`。
如果`Child`没有`watch`，返回一个新对象，新对象的原型上有`Parent`的`watch`；
如果`Parent`没有`watch`，返回`Child`的`watch`，返回之前assert`Child`的`watch`是一个对象；
遍历`Child`的`watch`里所有key和value，合并在一起；
总之`watch`是子元素和父元素如果有相同的key是通过`parentVal.concat(childVal)`返回一个新数组后给子元素
举个例子：
```
var Parent = Vue.extend({
  watch: {
    parent: function parent() {},
    both  : function bothFromParent() {},
  }
})
var Child = Parent.extend({
  watch: {
    child: function child() {},
    both : function bothFromChild() {},
  }
})
```
看下结果：
{% asset_img merge_watch_result_devtools.png %}

## 合并`props`, `methods`, `inject`和`computed`

```
strats.props =
strats.methods =
strats.inject =
strats.computed = function (
  parentVal: ?Object,
  childVal: ?Object,
  vm?: Component,
  key: string
): ?Object {
  if (childVal && process.env.NODE_ENV !== 'production') {
    assertObjectType(key, childVal, vm)
  }
  if (!parentVal) return childVal
  const ret = Object.create(null)
  extend(ret, parentVal)
  if (childVal) extend(ret, childVal)
  return ret
}
```
这个策略就是简单的通过2个extend函数，先把父元素的值赋值到新对象上，再遍历子元素，把子元素的值覆盖到新对象上。
## 合并`provide`
`provide`的合并和data类似
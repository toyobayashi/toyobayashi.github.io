---
title: 用 Vue 3 的写法写 React
date: 2020-12-13 16:21:05
tags: ['学习笔记']
---

react + @vue/reactivity = reactivuety

<!-- more -->

# 关于 @vue/reactivity

Vue 3 的核心响应式实现都在这个库里。

## ref

`ref()` / `shallowRef()` 返回一个 `RefImpl`。

### RefImpl 类

* `_rawValue`：原始值

* `_value`：如果创建的是 shallow ref 那么就是原始值，如果不是 shallow 且原始值为对象，这个成员保存 `reactive()` 转换后的原始对象 `Proxy`，否则也是原始值

* `getter value()`：依赖收集后返回 `_value`

* `setter value(newValue)`：对比 `newValue` 和 `_rawValue` 是否浅相等，不等则按上述规则改变 `_rawValue` 和 `value`

## reactive

`reactive()` / `shallowReactive()` / `readonly()` / `shallowReadonly()` 返回一个 `Proxy`。

## computed

`computed()` 返回一个 `ComputedRefImpl`，和 `RefImpl` 差不多，如果不传 setter，则不可以更改值。

## effect

重头戏。`effect(fn, options)` 返回一个包装 fn 后的 runner 函数，如果设置 `options.lazy` 为 false 或不设置，fn 立即执行，如果设置为 true，可以自行手动调用 runner 执行 fn。fn 里的 ref / reactive 如果被访问过值就会被依赖收集，被收集过的响应式对象如果值变了，`options.scheduler` 就会执行，如果没有设置 `options.scheduler`，那么 fn 本身就是 scheduler。

## stop

`stop()` 传入 `effect` 返回的 runner，停止监听变化，当响应式对象的值变了后 scheduler 不会再执行。

# 集成到 react

先看一个 Vue 3 计数器例子：

``` html
<template>
  <div>
    <div>{{count}} * 2 = {{doubleCount}}</div>
    <button @click="add"></button>
  </div>
</template>
```
``` js
import { ref, computed, defineComponent } from 'vue'

export default defineComponent({
  setup () {
    const count = ref(0)
    const doubleCount = computed(() => count.value * 2)
    const add = () => { count.value++ }

    return { count, doubleCount, add }
  }
})
```

实现用 react 来这样写：

```jsx
export default (props) => {
  const { count, doubleCount, add } = useSetup((propsProxy) => {
    const count = ref(0)
    const doubleCount = computed(() => count.value * 2)
    const add = () => { count.value++ }

    return { count, doubleCount, add }
  }, props)

  return (
    <div>
      <div>{count.value} * 2 = {doubleCount.value}</div>
      <button onClick={add}></button>
    </div>
  )
}
```

useSetup hook 传入 setup 函数，返回响应式对象，直接写在 JSX 上。

## 第一步：forceUpdate

这样写意味着组件的状态更新全部交给 vue 的响应式系统来处理，自然要使用 forceUpdate 强制更新组件。

```js
export function useForceUpdate () {
  const setState = useState(Object.create(null))[1]
  return useCallback(() => { setState(Object.create(null)) }, [setState])
}
```

## 第二步：依赖收集

setup 函数返回的响应式对象都要被依赖收集，所以要把收集的过程丢进 effect 里执行，并且只执行一次，大概是这样

```js
import { useRef, useCallback } from 'react'
import { effect, reactive, readonly } from '@vue/reactivity'

export function useSetup (setup, props) {
  // ...
  const forceUpdate = useForceUpdate()
  const instanceRef = useRef()

  const updateProps = useCallback(() => {
    if (instanceRef.current) {
      const keys = Object.keys(props)
      for (let i = 0; i < keys.length; i++) {
        const key = keys[i]
        instanceRef.current.props[key] = props[key]
      }
    }
  }, [props])

  if (!updateProps.__called) {
    updateProps()
    updateProps.__called = true
  }

  const updateCallback = useCallback(() => {
    // 生命周期 onBeforeUpdate
    forceUpdate()
    // 生命周期 onUpdated
  }, [])

  if (!instanceRef.current) {
    const reactiveProps = reactive({ ...props })
    const readonlyProps = readonly(reactiveProps)
    instanceRef.current = { /* ... */ }
    // ...
    let runner = null
    let ret
    try {
      ret = setup(readonlyProps)
    } catch (err) {
      // 其它错误处理
      throw err
    }
    runner = effect(() => {
      traverse(ret)
    }, {
      lazy: true,
      scheduler: () => {
        queueJob(updateCallback) // 入队，本轮同步代码全部执行完后在下一个 tick 更新组件
      },
      // ...
    })
    runner()
    // 生命周期 onBeforeMount
  }
  
  useEffect(() => {
    // 生命周期 onMounted
    return () => {
      // 生命周期 onUnmounted
    }
  }, [])

  return ret
}
```

收集过程就是递归遍历访问值。

```js
function traverse (value, seen = new Set()) {
  if (!isObject(value) || seen.has(value)) {
    return value
  }
  seen.add(value)
  if (isRef(value)) {
    traverse(value.value, seen)
  } else if (isArray(value)) {
    for (let i = 0; i < value.length; i++) {
      traverse(value[i], seen)
    }
  } else if (isSet(value) || isMap(value)) {
    value.forEach((v) => {
      traverse(v, seen)
    })
  } else {
    for (const key in value) {
      traverse(value[key], seen)
    }
  }
  return value
}
```

完整实现请移步 [toyobayashi/reactivuety](https://github.com/toyobayashi/reactivuety)。

Feature：

* `useSetup(setup, reactProps)` react hook, setup 函数可返回对象或 render 函数。

* `defineComponent(setup, reactRender?)` 

* `<Input>` `<Select>` `<Textarea>` 的 vModel 双向绑定

* `watch` 和 `watchEffect`

* 八个生命周期钩子

* `ref` 兼容 react 的 ref

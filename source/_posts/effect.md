---
title: effect
date: 2020-04-27 23:16:50
tags:
---

## effect是一个函数

```javascript

const num = reactive({currentNum:0})
let changeValue
const callBackFn = ()=>{ changeValue = num.currentNum }
effect(callBackFn)

```

## effect 是一个随着依赖变化而执行自定义函数的函数？

```javascript

const num = reactive({currentNum:0})
let changeValue
const callBackFn = ()=>{ changeValue = num.currentNum }
console.log(changeValue) // 0;
effect(callBackFn)
num.currentNum = 1;
console.log(changeValue) // 1;

```

## 它们是怎样勾搭一起的？

### effect 函数

```javascript

function effect<T = any>(
  fn: () => T,
  options: ReactiveEffectOptions = EMPTY_OBJ
): ReactiveEffect<T> {
  if (isEffect(fn)) {
    fn = fn.raw
  }
  const effect = createReactiveEffect(fn, options)
  if (!options.lazy) {
    effect()
  }
  return effect
}

```
effect 函数做了三步工作：
1. 是否已经被effect勾搭过
2. 创建effect勾搭过程
3. 是否默认执行effect函数


第一步分析省略...

### createReactiveEffect函数

```javascript

function createReactiveEffect<T = any>(
  fn: () => T,
  options: ReactiveEffectOptions
): ReactiveEffect<T> {
  const effect = function reactiveEffect(...args: unknown[]): unknown {
    return run(effect, fn, args)
  } as ReactiveEffect
  effect._isEffect = true
  effect.active = true
  effect.raw = fn
  effect.deps = []
  effect.options = options
  return effect
}

```

把目光移向 fn 变量，看看它被怎样处理：
1. 被放进一个 reactiveEffect 函数
2. 存放到一个叫 raw 的key
总的来说 createReactiveEffect 函数就是负责创建一个真正的 effect 函数的工作。但到现在还没有看到它们是怎么勾搭一起的，下面看看 run 函数。

## run函数

```javascript

function run(effect: ReactiveEffect, fn: Function, args: unknown[]): unknown {
  if (!effect.active) {
    return fn(...args)
  }
  if (!effectStack.includes(effect)) {
    cleanup(effect)
    try {
      effectStack.push(effect)
      activeEffect = effect
      return fn(...args)
    } finally {
      effectStack.pop()
      activeEffect = effectStack[effectStack.length - 1]
    }
  }
}

```

嗯！找到了！来！抓重点！看try-fially！

先分析try-finally的运行顺序：

1. effectStack.push(effect)
2. activeEffect = effect
3. fn(...args)
4. effectStack.pop()
5. activeEffect = effectStack[effectStack.length - 1]

activeEffect ，这是一个全局变量。

官宣：2-3序列，就是勾搭过程了。

### num.currentNum 与 fn 的关联在哪？

先来看看下面这段代码，是关于 Proxy 的 Get 阶段里的 track 函数：

```javascript

export function track(target: object, type: TrackOpTypes, key: unknown) {
  if (!shouldTrack || activeEffect === undefined) {
    return
  }
  let depsMap = targetMap.get(target)
  if (depsMap === void 0) {
    targetMap.set(target, (depsMap = new Map()))
  }
  let dep = depsMap.get(key)
  if (dep === void 0) {
    depsMap.set(key, (dep = new Set()))
  }
  if (!dep.has(activeEffect)) {
    dep.add(activeEffect)
    activeEffect.deps.push(dep)
    if (__DEV__ && activeEffect.options.onTrack) {
      activeEffect.options.onTrack({
        effect: activeEffect,
        target,
        type,
        key
      })
    }
  }
}

```

还记得 fn 函数里的 changeValue = num.currentNum 吗？

当 num 去获取 currentNum 的时候，触发了 Get ，而当前没有存储此 effect ，所以进入到这里：

```javascript

  if (!dep.has(activeEffect)) {
    dep.add(activeEffect)
    activeEffect.deps.push(dep)
    if (__DEV__ && activeEffect.options.onTrack) {
      activeEffect.options.onTrack({
        effect: activeEffect,
        target,
        type,
        key
      })
    }
  }

```

理所当然的被存储了，然后就是勾搭上了，因为每个key都会有对应的 dep 容器。

### num.currentNum产生变化，回调存取函数

建立了关联，只需要在需要的时候，执行对应的 dep 容器就可以了。可以了解一下 trigger 
若想了解详细的track、trigger的具体流程，请留意本系列相关文章，此处就忽略！

## effect第二个参数是什么？

从上面代码看到第二个options参数是一个对象，里面有：

```javascript

export interface ReactiveEffectOptions {
  lazy?: boolean
  computed?: boolean
  scheduler?: (run: Function) => void
  onTrack?: (event: DebuggerEvent) => void
  onTrigger?: (event: DebuggerEvent) => void
  onStop?: () => void
}

```

未完待续！
欢迎斧正！
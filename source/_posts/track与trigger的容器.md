---
title: track与trigger的容器
date: 2020-04-27 23:12:32
tags:
---

## 容器结构：

```javascript

WeakMap:{
  Object: Map(ObjectKey,Set([]))
}

```

## 结构图片
 
 ![](https://cdn.nlark.com/yuque/0/2020/png/254136/1578902229074-ffd2e78d-8529-468b-8488-66894b897f69.png)

• WeakMap内部 紫色区域
• Map 灰色区域
• ObjectKey 粉色区域
• Set 黄色区域

## 问题的思考：

### 为什么用WeakMap？

1、WeakMap键名所引用的对象都是弱引用！而且不在JavaScript垃圾回收机制不将其引用考虑在内。就是说，没有引用就自动被清除。

```javascript

let weak = new WeakMap();
let obj = { a: { a: 1 } };
let otherObj = obj;
let objSubRef = obj.a;
weak.set(obj, "存放");
obj = null;
console.log(weak.get(obj)); // undefined
console.log(weak.get(otherObj)); // 存放
otherObj = null;
console.log(weak.get(otherObj)); // undefined
console.log(objSubRef); // {a:1}
//若在垃圾回收机制中，objSubRef 存在 obj 里的引用，垃圾机制是不回收的。

```

2、WeakMap只接受对象为key，每个数据的引用都是唯一，解决重复key的可能，可以统一存取。
为什么WeakMap的value是用Map来存取？

开发过程中，key不只是String，有可能是symbol等等。详情点击此处

### 为什么Map的value是用Set来存取？

1、存回调函数，Vue-3.0的 effect API 了解一下。 
2、 Set的遍历顺序是按照插入顺序来运行。详情点击此处

## 容器的收集机制

![](https://cdn.nlark.com/yuque/0/2020/png/254136/1580733071459-0c7d1682-bedb-4dc3-ba69-26435e888348.png)

### track

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
track的
主要集中为两个方向服务：

1. 容器的初始化与数据收集。

```javascript

  let depsMap = targetMap.get(target)
  if (depsMap === void 0) {
    targetMap.set(target, (depsMap = new Map()))
  }
  let dep = depsMap.get(key)
  if (dep === void 0) {
    depsMap.set(key, (dep = new Set()))
  }

  ```
2. 当前key是否与 effect API 组合使用？若是：把 effect 包裹的函数收集； effect 第二个参数是否有参，且为onTrack，若有，立刻调用。

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
### trigger

```javascript

export function trigger(
  target: object,
  type: TriggerOpTypes,
  key?: unknown,
  extraInfo?: DebuggerEventExtraInfo
) {
  const depsMap = targetMap.get(target)
  if (depsMap === void 0) {
    // never been tracked
    return
  }
  const effects = new Set<ReactiveEffect>()
  const computedRunners = new Set<ReactiveEffect>()
  if (type === TriggerOpTypes.CLEAR) {
    // collection being cleared, trigger all effects for target
    depsMap.forEach(dep => {
      addRunners(effects, computedRunners, dep)
    })
  } else {
    // schedule runs for SET | ADD | DELETE
    if (key !== void 0) {
      addRunners(effects, computedRunners, depsMap.get(key))
    }
    // also run for iteration key on ADD | DELETE
    if (type === TriggerOpTypes.ADD || type === TriggerOpTypes.DELETE) {
      const iterationKey = isArray(target) ? 'length' : ITERATE_KEY
      addRunners(effects, computedRunners, depsMap.get(iterationKey))
    }
  }
  const run = (effect: ReactiveEffect) => {
    scheduleRun(effect, target, type, key, extraInfo)
  }
  // Important: computed effects must be run first so that computed getters
  // can be invalidated before any normal effects that depend on them are run.
  computedRunners.forEach(run)
  effects.forEach(run)
}

```

主要为目标对应dep容器存储的函数的执行:
1. 筛选computer组、普通函数组。
2. 执行分先后，先computer组，后普通函数组。
欢迎斧正！

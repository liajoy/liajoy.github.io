---
title: 一道 Typescript 笔试题的解答
date: 2020-03-26 11:43:22
tags:
- typescript
---
最近看见一道有趣的 Typescript 闭上题，题目地址在[这儿](https://github.com/LeetCode-OpenSource/hire/blob/master/typescript_zh.md)。

地址中关于题目的描述已经很清楚啦，提取一下要点，主要是两个问题：

- 如何提取一个包含方法与属性的对象中的方法
- 如何获取泛型类型

好了，问题清楚了，那就来看看如何解决它们。

## 如何提取对象中的方法

首先，我们先筛选出方法：

``` typescript
type MethodNameMap<T> = {
    [K in keyof T]: T[K] extends Function
        ? K
        : never
}

type Obj = {
    a: number,
    b: () => void,
    c: () => void
}
type ObjMethodNameMap = MethodNameMap<Obj> // { a: never, b: 'b', c: 'c' }
```

然后，把它变成一个联合类型：

``` typescript
type MethodNames<T> = MethodNameMap<T>[keyof MethodNameMap<T>]

type ObjMethodNames = MethodNames<Obj> // 'b' | 'c'
```

这样，我们就拿到了所有的方法名了，如果还需要拿到方法的类型，只需要 `Pick` 就完事了：

``` typescript
type PickMethods<T> = Pick<T, MethodNames<T>>

// Test
type ObjMethods = PickMethods<Obj> // { a: () => void, b: () => void }
```

## 如何获取泛型类型

Typescript 中有一个 `infer` 关键字，它可以来指代它所在位置的类型，比如：

``` Typescript
type ArrayItem<T> = T extends Array<infer U> ? U : never

type NumArray = Array<number>
type Num = ArrayItem<NumArray> //number
```

通过这个能力我们可以很轻松的完成泛型类型提取，比如提取 `Promise` 的返回值：

``` typescript
type PromiseResolve<T> =
    T extends Promise<infer U> ? U :
    never
```

## 解题

解决了这两个问题之后，我们再回过头来看这道题。

> 现在有一个叫 connect 的函数，它接受 EffectModule 实例，将它变成另一个一个对象，这个对象上只有EffectModule 的同名方法。

针对这个问题，通过上文的 `methodNames` 即能拿到方法名。

> 方法的类型签名被改变了:
>
> `asyncMethod<T, U>(input: Promise<T>): Promise<Action<U>> ` 变成了 `asyncMethod<T, U>(input: T): Action<U> `
>
> `syncMethod<T, U>(action: Action<T>): Action<U>` 变成了
> `syncMethod<T, U>(action: T): Action<U>`

针对这个问题，通过 infer 解决。

最后，这道题的 `Connect` 的类型就是这样了：

``` typescript
type EffectModuleMethodNames = MethodNames<EffectModule>
type Connect = (module: EffectModule) => {
    [K in EffectModuleMethodNames]:
        EffectModule[K] extends (input: Promise<infer T>) => Promise<infer U> ? (input: T) => U :
        EffectModule[K] extends (action: Action<infer T>) => infer U ? (input: T) => U :
        never
}
```

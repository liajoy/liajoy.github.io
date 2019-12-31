---
title: Optional chaining
date: 2019-04-28 17:31:02
tags:
- Javascript
---

[Optional chaining](https://github.com/tc39/proposal-optional-chaining) 是 ES2020 的新语法，目前处于 stage 4 阶段。在介绍它之前，先来看看这样的一个数据结构：

``` typescript
interface Person {
    body?: {
        hand?: {
            finger?: number
        }
    }
}
```

如果我们需要获取 `finger` 的值，可能会这么做：

```javascript
let finger
if(person) {
    if(person.body) {
        if(person.body.hand) {
            if(person.body.hand.finger) finger = person.body.hand.finger
        }
    }
}
```

难以想象，一个现代的高级语言，为了从 `person` 中取到可选属性 `finger`，这么简单的功能竟然要写这么多代码？于是，Optional chaining 诞生了。

## 没有 Optional chaining 是怎么处理

像上文那样大量的 if 判断是不符合程序员审美的，那么目前有哪些解法呢？

一个方法是把 if 优化成 && 操作符，像这样：

``` javascript
const finger = (
    person &&
    person.body &&
    person.body.hand &&
    person.body.hand.finger
)
```

虽然代码量还是不少，不过比起 if 判断已经是很大的改进了。

也可以使用 `lodash.get`，像这样：

``` javascript
const finger = lodash.get(person, 'body.hand.finger')
```

似乎许多人都是采用这个方法，不过由于不支持代码提示，个人不喜欢这个方法。

第三个是 `try...catch`：

``` javascript
let finger
try {
    finger = finger.body.hand.finger
} catch() {}
```

这个用法很「非常规」，不过排除个人喜好不谈，它的缺点我也不知道在哪。如果怀疑可能的性能问题，可以参考 [benchmark](https://jsperf.com/accessing-property-chains)。

除此之外，通过使用 Proxy 代理出一个新对象也能实现：

``` javascript
const optionalChaining = defaultVal => (
    new Proxy(() => defaultVal, {
        get: (target, key) => optionalChaining((defaultVal || {})[key])
    })
)

const finger = optionalChain(person).body.hand.finger()
```

既支持智能提示，代码量又小，然而就「取值时必须调用函数」，这个操作就够让人排斥了。理论上，通过 Proxy 我们可以实现不少元编程的特性，然而事实上并不是那么简单，例如 `ownKeys` trap，我们随意返回一个包含不存在于对象上的 key 的数组，是无效的。

``` javascript
const obj = {
    a: 1
}
const proxy = new Proxy(obj, {
  ownKeys (target) {
    return ['a', 'b', 'c']
  }
});
console.log(Object.keys(proxy)) // ['a']
```

虽然方法不少，不过就目前来说没有一个比较完美的解决方案，那么回到主角 optional chaining 上来。

## Optional chaining

Optional chaining 的语法很简单，它的常见用法如下（摘自草案）：

```javascript
a?.b // a && a.b
a == null ? undefined : a.b

a?.[x] // a && a[x]
a == null ? undefined : a[x]

a?.b() // a && a.b()
a == null ? undefined : a.b()

a?.() // a && a()
a == null ? undefined : a()

delete a?.b
a == null ? true : delete a.b
```

语法很简单，草案里关于它的所有要注意的点都说明得很详细，非常建议看草案。提一下几个有意思的点：

- 短路：如 `a?.[++x]` 这样的情况，当 `a == null` 时，`++x` 是不会执行的。
- 为什么不使用 `a?b?c` 而是 `a?.b?.c`：想象一个这样的函数调用 `a.b?(10):2`，引擎需要如何解释它呢？

## Can i use

~~VSCode 不支持，据说 WebStorm 支持（没试过）。~~
~~理论上，VsCode 中可以关闭 javascript Validate，完全使用 babel 进行校验，然而这样得不偿失~~

TS 更新到 3.7 以后可以使用这个特性，VSCode 在 1.41 也支持啦！现在已经可以愉快的使用这个特性了。

## 参考

- [Optional Chaining for JavaScript](https://github.com/tc39/proposal-optional-chaining)

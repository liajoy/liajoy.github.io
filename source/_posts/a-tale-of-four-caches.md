---
title: 「译」关于四种缓存的故事
date: 2018-03-12 00:12:15
tags:
- 计算机基础
- 网络
---
> 原文地址：[A Tale of Four Caches](https://calendar.perfplanet.com/2016/a-tale-of-four-caches/)

- - -

最近 preload、HTTP/2 push 以及 Service workers 中与浏览器缓存有关的部分受到了不少讨论，许多人对它们都有或多或少的疑惑。

因此，我想给你们讲述一个故事，故事的主人公是一个 HTTP 请求，它肩负着找到与其匹配的资源的使命，为了达成这个使命而旅行。

```
这个故事基于 Chromium 的术语与概念，在其他浏览器可能会有所不同。
```

## Questy 的旅行

Questy 是一个请求。它由渲染引擎（简称 renderer）创建，它渴望找到一个让它达成使命并且永远（至少到由于标签被关闭而导致的文档分离前）快乐地在一起的资源。

![Questy，向往着它的资源](https://user-gold-cdn.xitu.io/2018/3/12/16217fb247b61a9a?w=800&h=666&f=webp&s=14724)

于是 Questy 启程去追寻它的幸福。那么，它能在哪里找到合适的资源呢？

最近的地方是...

## 内存缓存

内存缓存里有一个充满资源的巨大容器，这个容器包含了 renderer 为当前文档所获取的一切资源，并且会在文档的生命周期内好好保存这些资源。这就意味着如果 Questy 寻找的资源已经在当前文档的其他地方被获取过，那么它将会在内存缓存中找到这个资源。

不过称它为「短期内存缓存」或许更合适，因为内存缓存只在导航结束前保存资源，在某些情况下还可能更短。

![短期内存缓存和它的容器](https://user-gold-cdn.xitu.io/2018/3/12/1621809602387a3f?w=800&h=516&f=webp&s=12728)

有许多原因会导致 Questy 所寻找的资源已经被获取过这一现象。

[preloader](https://calendar.perfplanet.com/2013/big-bad-preloader/) 可能最常见的一个。如果 Questy 是作为 DOM 节点被 HTML 解析器创造出来，那么在 preloader 的 HTML 标记化阶段中，它所需要的资源很有可能已经被获取了。

另一个原因是，显式的 [preload](https://www.smashingmagazine.com/2016/02/preload-what-is-it-good-for/) 的指令(`<link rel=preload>`) 也会导致这一现象。

除此之外，先前的 DOM 节点或者 CSS 规则也可能已经获取了该资源。例如，一个页面包含多个 `src` 属性相同的 `<img>` 元素，这时只会获取一个资源。这种能够让多个元素获取一个资源的机制就是源于内存缓存的存在。

但是内存缓存并不会轻易让请求匹配到资源。显然，为了让请求和资源匹配，他们不仅需要有相匹配的 URL，还必须匹配资源类型，CORS 模式和一些其他特性。

![](https://user-gold-cdn.xitu.io/2018/3/12/162183084537727a?w=1600&h=674&f=webp&s=29826)

> 来自内存缓存的请求的匹配特征在规范中并没有很好的定义，因此在浏览器实现中可能会略有不同。

有一个情况是，内存缓存不关心 HTTP 语义。就算存储的资源的消息头中设置了 `max-age=0` 或 `no-cache` `Cache-Control` ，内存缓存也不会关心它们的作用。由于它允许在当前导航中重用资源，所以 HTTP 语义在这里并不重要。

![](https://user-gold-cdn.xitu.io/2018/3/12/162183144e4c4596?w=1600&h=654&f=webp&s=32258)

有一个例外是 `no-store` 指令，内存缓存在某些情况下会遵守该指令（例如，当资源被单独的节点重用时）。

Questy 继续向内存缓存寻求匹配的资源，不过并没有找到。

但 Questy 并没有放弃。它达到了 Resource Timing 和 DevTools network 注册点，在那里它被注册为一个寻找资源的请求（这意味着它现在将显示在 DevTools 以及 resource timing 中，假定它最终会找到资源）。

完成注册后，它继续朝着...

## Service Worker 缓存

与内存缓存不同，Service Worker 缓存并不遵循任何传统规则。它只遵守他们的主人（即 Web 开发者）告诉它的规则。因此在某种程度上它是不可预测的。

![努力工作的 Service Worker](https://user-gold-cdn.xitu.io/2018/3/12/162183ef199ec3d6?w=800&h=988&f=webp&s=17818)

首先，只有当页面安装了 Service Worker，它才会存在。而且由于它的逻辑不是内置于浏览器，而是由 Web 开发者通过 JavaScript 定义的，Questy 并不知道它是否愿意帮自己寻找资源，即便它愿意，那个资源就会是它所寻找的资源吗？还是只是由 Service Worker 的主人的奇怪逻辑所创建的一个假响应？

没有人知道。因为 Service Workers 拥有自己的逻辑，所以他们可以自由地完成匹配请求和潜在资源、包装 Response 对象这些行为。

Service Worker 有一个使它能够保留资源的 Cache API。它和内存缓存的一个主要区别是它是持久的。即使选项卡关闭或浏览器重新启动，存储在该缓存中的资源仍会保留。有一种情况会导致缓存的资源被逐出，即开发者明确将他们逐出（使用`cache.delete(resource)`）。另一种情况是，当浏览器用完了存储空间，整个 Service Worker 缓存会与所有其他原始存储，如 indexedDB、localStorage 等，一起被删除。这样，Service Worker 就保持在它缓存中的资源与它自身以及其他原始存储之间同步。

Service Worker 只会负责最多一个 host 的范围。因此 Service Worker 只能对该范围内的文档请求进行响应。

Questy 找到了 Service Worker 并问它是否有为自己准备的资源。但 Service Worker 没有在自己领域内的看到它要的资源，所以没有资源能给 Questy。因此，Service Worker 让（使用`fetch()`）Questy 继续在网络堆栈的未知大陆上搜索资源。

而在网络堆栈中，寻找资源的最好地方就是...

## HTTP 缓存

HTTP 缓存，有时也被它的缓存朋友称为「磁盘缓存」，它与 Questy 之前看到的缓存完全不同。

一方面，它是持久的，允许资源在会话之间甚至跨站点重用。如果某个资源由一个站点缓存，那么 HTTP 缓存也允许其他站点重用该资源。

同时，HTTP 缓存遵循 HTTP 语义（它的名字就表明了这一点）。它乐于为其认为「新鲜」的资源提供服务（基于缓存生命周期，由其响应的缓存头指示），重新 [验证](https://www.mnot.net/cache_docs/#VALIDATE) 资源，并拒绝存储不该存储的资源。

![十分严格的 HTTP 缓存](https://user-gold-cdn.xitu.io/2018/3/12/16219029dfa64684?w=765&h=833&f=webp&s=17226)

它是一个持久缓存，所以它也需要驱逐资源，但与 Service Worker 缓存不同的是，只要缓存需要空间去储存更重要或更流行的资源时，资源就能一个接一个地被逐出。

HTTP 缓存具有一个基于内存的组件。在组件中，它会对进入的请求进行资源匹配。但当它找到了匹配的资源时，它会从磁盘中获取资源内容，而这会是一个昂贵的操作。

```
我们之前提到过，HTTP 缓存尊重HTTP语义。这个说法在大多数情况下都是正确的，但有一个例外：HTTP 缓存会在有限的时间内存储资源。
通过显式的指令（`<link rel=prefetch>`）或者浏览器的内部策略，能为下一个导航预获取资源，而那些预获取的资源会保存到下次导航，即使它们是不可缓存的。
所以当这种预获取的资源到达 HTTP 缓存时，它会被缓存（并无需重新验证）5分钟。
```

HTTP缓存看起来相当严格，但 Questy 还是鼓起勇气问它是否有匹配的资源，答案仍然是没有。

于是它将不得不继续走向网络。通过网络的旅程是可怕且不可预知的，但 Questy 明白它无论如何都必须找到它的资源。所以它继续前进。它发现了一个 HTTP/2 会话，接着很快就会通过网络发送，这时它突然看到了......

## Push 「Cache」

Push 缓存（称为「unclaimed push streams container」或许更合适，但不那么容易理解）是存储 HTTP/2 推送资源的地方。它们作为 HTTP/2 会话的一部分进行存储，并具有多种含义。

![「unclaimed push streams container」，即 Push 缓存](https://user-gold-cdn.xitu.io/2018/3/12/162190f44588e3db?w=800&h=883&f=webp&s=32830)

这个容器没有任何持久性。如果会话被终止，那么所有没有被请求的资源都会消失。同时，如果使用不同的 HTTP/2 会话获取资源，它将 [不会匹配](https://bugs.chromium.org/p/chromium/issues/detail?id=669515) 。最重要的是，资源只会在有限的时间内保存在 push 缓存容器中，在基于 Chromium 的浏览器中这个时间大概是 5 分钟。

push 缓存根据其 URL 以及各种请求头匹配请求与资源，但它不适用严格的 HTTP 语义。

```
push 缓存在规范中也没有很好的定义，实现可能因浏览器、操作系统和其他 HTTP/2 客户端而异。
```

Questy 虽然没报太大希望，但它仍然询问 Push 缓存是否有匹配它的资源。而令人惊讶的是，它有资源！Questy 非常开心地接受了资源（这意味着它从无人认领的容器中删除了 HTTP/2 流）。现在它可以带着资源回到 renderer 中去了。

在他们返回的路上，被 HTTP 缓存滞留，HTTP 缓存拿了一份资源的拷贝存储着，以防将来的请求需要用到。

当他们离开网络堆栈返回到 Service Worker 中时，Service Worker 也储存了一份资源拷贝，之后再送他们回到 renderer。

终于，他们回到了 renderer，内存缓存保留了资源的引用（不是拷贝），以便在这个导航会话中为将来的请求分配相同的资源。

![](https://user-gold-cdn.xitu.io/2018/3/12/162192325b93cddc?w=1600&h=1509&f=webp&s=60024)

他们从此过着幸福快乐的生活。不过当文档被分离时，他们都会去见垃圾回收器。

![](https://user-gold-cdn.xitu.io/2018/3/12/1621924e7c3a4fce?w=725&h=985&f=webp&s=19602)

但那是另一天的故事了。

## 结论

那么，我们能从 Questy 的旅程中学到什么？

* 不同的请求可以与浏览器的不同缓存中的资源进行匹配。
* 与请求资源所匹配的缓存可能会影响 DevTools 和 Resource Timing 中显示的方式。
* 推送的资源不会永久存储，除非它们的流被请求接受。
* 不可缓存的预加载资源将不会用于下一个导航。这是 preload 和 prefetch 的主要区别之一。
* 这里边还有许多不明确的地方，观察到的行为可能会因浏览器实现而有所不同。而这是我们需要解决的问题。

总而言之，如果你使用 preload、H2 push、Service Worker 或其他先进技术来尝试加速你的网站时，你可能会注意到内部缓存实现的情况。通过了解这些内部缓存以及它们的运行方式可能会帮助你更好地理解网站现状，并有可能避免不必要的问题。

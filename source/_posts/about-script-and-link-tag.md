---
title: script 与 link 标签位置的影响
date: 2017-12-29 18:39:31
tags:
- 浏览器基础
---
老生常谈的话题了，我们都知道 `script` 标签要放在 `body` 标签的尾部， `link` 标签需要放在 `head` 标签里，但是为什么呢？

在开始之前，先来看看 [浏览器渲染流程](https://calendar.perfplanet.com/2012/deciphering-the-critical-rendering-path/):

![document render steps, with JavaScript](http://www.igvita.com/posts/12/doc-render-js.png)

从图中可以看到，我们得到与本文相关的关键信息：

- 在 Layout 前需要得到 RenderTree，而 RenderTree 是由 DOM 与 CSSOM 组成。
- JavaScript 会影响 DOM 与 CSSOM。

那么进入正题，我们选取几个常规的 `link` 与 `script` 标签放置的位置：

- `head` 中
- `body` 中
- `body` 尾

## `link` 标签

- `link` 标签放置在 `head` 中时，浏览器会等待所有外链样式加载完再进行渲染。
- `link` 标签放置在 `body` 中时，浏览器会渲染 `script` 标签与 `link` 标签前的内容，直到上述标签的资源加载完成时，剩下的内容才会渲染。
- `link` 标签放置在 `body` 尾与放在 `body` 中一样。

## `script` 标签

- `script` 标签放置在 `head` 中时，浏览器会等待 `script` 下载并执行完再渲染，所以在执行之前页面是空白的。
- `script` 标签放置在 `body` 中时，浏览器会渲染 `script` 标签与 `link` 标签前的文档内容，直到上述标签的资源加载完成时，剩下的内容才会渲染。
- `script` 标签放置在 `body` 尾与放在 `body` 中一样，只是由于其后已经没有文档内容了，所以一切正常。

这里有例外是，当给 `script` 标签设置 `defer` 或 `async` 属性后，上述的结论就不成立了，它们对文档解析的影响如下图：

![What's the difference between async vs defer attributes](https://res.cloudinary.com/josefzacek/image/upload/v1520507339/blog/whats-the-difference-between-async-vs-defer-attributes.jpg)

## 总结

- `script` 标签与 `link` 标签不会影响文档资源的加载，也就是说资源的加载与文档的解析没有相关性。
- `script` 标签与 `link` 标签会影响文档的解析，所以位于其后的文档内容不会渲染，为什么呢？因为这两个资源有可能会导致 reflow 与 repaint。
- `script` 标签放在 `body` 尾部是因为不希望它阻塞文档的解析与影响渲染结果。
- `link` 标签放在 `head` 中是因为不希望页面先渲染出不带样式的结果后再渲染出正确的结果。

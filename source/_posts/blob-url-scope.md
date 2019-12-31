---
title: 被忽略的 blob URL
date: 2019-12-31 22:07:28
tags:
- 浏览器基础
---
blob URL 是大家都不陌生的一个特性，对于它，我一直以来都没有特意去了解细节。直到最近同事跟我打赌说「blob URL 只能在当前文档（blob URL 被创建的文档）中使用」，我心想，一直以来我都是在新标签页预览 blob URL，怎么可能只能在当前文档使用呢。虽然很笃定但是也很怂只赌了一顿午餐，于是立马动手实验并且成功得赢了一餐饭。

但是在实验的过程中，突然意识到 blob URL 中包含了当前文档的源的信息，既然它能跨文档使用，那么这个源是做什么用呢？于是就去补了 MDN 上的相关文档，然而并没有说明。无奈之下只能去看 W3C 标准。

## blob URL 中的源

在 W3C 标准中规定，通过 `revokeObjectURL(url)` 销毁 blob URL 时，将会执行以下操作：

1. 令解析后的 url 作为 url record。
2. 如果 url record 的 scheme 不为「blob」则返回。
3. 令 origin 为 url record 的 origin。
4. 令 settings 为 [current settings object](https://html.spec.whatwg.org/multipage/webappapis.html#current-settings-object)。
5. 如果 origin 与 settings 的 origin 不同则返回。
6. 删除 Blob URL Store 中的 url 入口

可以看到，当我们销毁 Blob URL 时，如果当前文档的源与 Blob URL 的源不一致，那么 URL 的销毁操作是会是失败的。

## blob URL 到底能不能跨文档

关于这个问题，没有在 W3C 标准中找到明确的说明，但是在 [这个](https://www.w3.org/TR/FileAPI/#creating-revoking) 部分中的 EXAMPLE 9 后有一段说明：

> Since a user agent has one global blob URL store, it is possible to revoke an object URL from a different window than from which it was created. The `URL.revokeObjectURL()` call ensures that subsequent dereferencing of myurl results in a the user agent acting as if a network error has occurred.

这里提到了每一个浏览器（user agent）中都有一个全局的 blob url store，也就是说这个 store 并不被只限制在当前的文档中。

## 参考

- [A URL for Blob and MediaSource reference](https://www.w3.org/TR/FileAPI/#url)
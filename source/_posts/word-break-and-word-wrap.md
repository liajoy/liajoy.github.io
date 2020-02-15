---
title: 关于 word-break 与 word-wrap 需要了解的知识
date: 2020-02-10 20:59:17
tags:
- CSS
---

CSS 中有一对容易让人混淆的规则，`word-break` 与 `word-wrap`。我们常常用它们进行换行的控制，不过我基本是处于用完就忘的状态，这次趁着机会总结一下关于它们的知识。

## `word-break`

在介绍它之前，我们先理解一个名词，soft wrap opportunity。soft wrap opportunity 指的是能够换行的断点位置，例如在大多数的书写系统里，空格和标点符号就是 soft wrap opportunity，排版时会在这些位置进行换行。

CSS 中的 `word-break` 就是用来定义 soft wrap opportunities。它支持这 4 个值，`normal`、 `break-all`、`keep-all`、`break-word`，默认值是 `normal`。

- `normal`，使用默认的换行规则。
- `break-all`，为了防止溢出，单词可以在任意为止被拆分来满足换行（中文、日文、韩文除外）。
- `keep-all`，CJK（中文、日文、韩文）中的「单词」不会被拆分，非 CJK 会使用 `normal` 的规则。
- `break-word`（**deprecated**），与设置了 `word-break: normal` 和 `overflow-wrap: anywhere` 具有同样的表现。

`break-all` 与 `keep-all` 的区别在于 CJK 中的「单词」是否会被拆分，可我寻思着长这么大也没见过中文里有单词这东西呀。事实上，这个单词其实就是标点符号或空格前的部分。具体表现如下：

`word-break: normal` 可被拆分

![word-break: normal](/images/word-break-normal.png)

`word-break: keep-all` 不可被拆分

![word-break: keep-all](/images/word-break-keep-all.png)

## `word-wrap`（`overflow-wrap`)

`word-wrap` 用来解决**不可分割文字**溢出的情况，它支持这 3 个值，`normal`、 `break-word`、`anywhere`，默认值是 `normal`。

- `normal`，在正常的换行点换行，例如两个单词中的空格。
- `anywhere`，为防止溢出，表示如果行内没有多余的地方容纳该单词，则那些正常的不能被分割的单词会被强制分割换行（如长字或URL）。在断点处没有插入连字符。但是在计算最小内容内在尺寸（min-content intrinsic sizes）时会考虑 `anywhere` 所产生的 soft wrap opportunities。
- `break-word`，同 `anywhere`，但是在计算最小内容内在尺寸（min-content intrinsic sizes）时**不会**考虑 `break-word` 所产生的 soft wrap opportunities。

另外，`word-wrap` 被重命名为 `overflow-wrap`。也是啊，单看 `word-wrap`，实在是难以想到它是用来解决溢出。不过 `overflow-wrap` 的支持性并不如 `word-wrap`，不过差别不大，具体参见 [caniuse](https://caniuse.com/#search=overflow-wrap)。

在测试 `word-wrap` 表现的时候，发现 `anywhere` 在 chrome 中竟然是一个非法值，所以 MDN 的 demo 里 `anywhere` 好像表现得和描述不一样。于是只能去找备胎 firefox。

### `anywhere` 与 `break-word`

啥是 min-content intrinsic sizes？从 W3C 里看到，简单来说，min-content intrinsic sizes 就是不会导致文本溢出并且尽量换行的最小宽度。

那么计算最小内容内在尺寸（min-content intrinsic sizes）和 soft wrap opportunities 又又又什么关系呢。计算最小内容内在尺寸要求尽量换行，而 soft wrap opportunities 又决定着哪些地方能换行。所以 `anywhere` 与 `break-word` 的区别就在于，它们所产生的换行点是否会影响最小内容内在尺寸的计算。

当我们给文本容器设置 `width: min-content` 时，我们才能看出 `anywhere` 与 `break-word` 的区别。

`overflow-wrap: anywhere` 会影响计算最小内容内在尺寸的计算，所以容器的宽度会足够小。

![overflow-wrap: anywhere](/images/overflow-wrap-anywhere.png)

`overflow-wrap: break-word` 不会对容器宽度产生影响。

![overflow-wrap: break-word](/images/overflow-wrap-break-word.png)

## 总结

经过上文的介绍，`word-break` 与 `word-wrap` 除了都有 `word` 好像就没啥容易混淆的了。

从用途上来说，`word-break` 是用来定义 soft wrap opportunities，即哪些地方是能够换行的。而 `word-wrap` 是用来解决不可分割文本溢出容器的情况的。那么文本是否可分割是由什么控制的呢？显然就是 `word-break`，所以 `word-break` 的优先级是高于 `word-wrap` 的。当设置了 `word-break: break-all` 后，就不存在不可分割文本了，那么 `word-wrap` 属性自然就无效了。

在我们日常使用中，对于大多数现代浏览器来说，建议使用 `overflow-wrap` 替代 `word-wrap`。因为 `word-wrap` 与 `word-break` 这两看久了，又要眼花了。

首先，建议不使用 `word-break: break-word`，一是因为它已经不被推荐使用了，二是太容易和 `word-wrap: break-word` 混淆了，虽然大多数情况下它们的表现是一样的。

其次，建议不使用 `anywhere`，一是目前在 chrome 的表现奇怪，二是 `min-content` 用的不多，它会出现与 `break-word` 的不同的表现的场景并不多。

当我们要解决文本溢出的情况，那么我们能用的只有 `overflow-wrap: break-word` 与 `word-break: break-all`，这两个的区别从字段名就能区分，就是在换行时单词是否能被分割。如果涉及到 CJK 的话，选择对应的值就行了。

最后，为什么我要花这么多时间来理清这两规则的关系？

## 参考

- [inline-breaking](https://www.w3.org/TR/css-text-3/#line-breaking)
- [word-break](https://developer.mozilla.org/en-US/docs/Web/CSS/word-break)
- [overflow-wrap](https://developer.mozilla.org/en-US/docs/Web/CSS/overflow-wrap)
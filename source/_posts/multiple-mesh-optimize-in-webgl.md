---
title: WebGL 的多 Mesh 渲染优化
date: 2019-09-23 20:16:49
tags:
- WebGL
---
前段时间基于雪碧大佬的 [beam](https://github.com/doodlewind/beam) 封装了一个简单的上层渲染库「Tiga」，主要做了一些 Material 与 Geometry 的封装，以及简单的多 Mesh 渲染流程优化。在「Tiga」中的主要实践是围绕着以下两个目标进行的：

- 减少 gl 状态设置
- 减少 draw call

## 减少 gl 状态设置

「Tiga」会保存当前绘制的 mesh 信息，例如 shader、vertex、texture 等，当绘制下一个 mesh 时，将比较下一个 mesh 的信息与当前的 mesh 信息的差异，之后再设置这部分差异的状态。

除此之外，对于不透明的 mesh，我们可以对 mesh 进行一次排序，如：

- 拥有相同 shader 的 Mesh 放在一起
- 拥有相同 texture 的 Mesh 放在一起
- z 小的放在前面

这样可以最大限度地减少切换 shader 或者上传 texture 的次数。

![sort mesh in Tiga](/images/tiga-sort.png)

## 减少 draw call

当一组 mesh 具有相同的 shader 与 texture 时，我们可以合并它们的 geometry，这样就只需要一次 draw call。但是，这样有一个问题，我们需要尽量保证这组 mesh 相对静态，否则频繁的操作缓冲区也是一个性能消耗。例如，游戏「我的世界」中的大量地方方块的绘制，只需要在初始化时将所有地形方块合并就能有不小的性能提升。

由于时间问题，关于 WebGL 的实践就到此，未完待续。。。

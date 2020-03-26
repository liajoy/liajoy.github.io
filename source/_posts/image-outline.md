---
title: Canvas 中的透明图像描边
date: 2019-12-23 00:11:14
tags:
- 图像处理
- WebGL
- Canvas
---
图像描边是设计软件中常见的图像处理功能，在 Canvas 中有 `strokeText` 能够直接对文字进行描边，那么有没有一个 API 能够对图像进行描边呢？很遗憾，并没有。为什么这么简单的功能都没有？那我们要如何实现描边呢？这就让我们来看看有哪些方案能够实现描边效果。

## SVG 滤镜

既然 Canvas 没有内置，那万能的 SVG 有没有呢？SVG 里有许多有趣的滤镜，其中的 `feMorphology` 可以达到将某些元素进行「扩张」或者「腐蚀」的效果。我们可以用它实现 [文字描边](https://tympanus.net/codrops/2019/01/22/svg-filter-effects-outline-text-with-femorphology/)。那如果将它 [应用在图像上](https://codepen.io/liajoy/pen/KKwmwom) 呢？

![outline-by-svg-filter](https://user-gold-cdn.xitu.io/2020/3/10/170c0231abf0d8c5?w=1166&h=956&f=png&s=469162)

好吧，看起来效果和我们的需求相去甚远，只得放弃这个方案。

## 图像偏移

我们先选择一张简单的矩形图像，如果将它进行填充并复制 8 份，把这 8 张分别沿着上、下、左、右、左上、左下、右上、右下八个方向进行偏移，就能完成对矩形图像的描边。不过它的描边结果不「圆润」，如果复制更多份，比如 360 份，让图像往 360 个方向进行偏移不就能做出圆角了吗？让我们看看结果：

![outline-by-offset](https://user-gold-cdn.xitu.io/2020/3/10/170c4cd52549042f?w=1112&h=1050&f=png&s=569987)

不过这个方案有着不少缺点：

  1. 耗时长，以一张 2000 * 2000px 的图像为例，在 Chrome 下完成一次描边需要 150ms 左右，而在 firefox 下需要 1s ，这也就意味着我们可能无法实时应用描边。
  2. 当描边的宽度超过了实际的图像尺寸后会出现镂空的现象，所以在描边宽度与图像尺寸上有限制。就像这样：
  ![outline-by-offset-bad-result](https://user-gold-cdn.xitu.io/2020/3/10/170c026bb40f8e1b?w=1274&h=1210&f=png&s=437762)
  3. 无法实现内描边。

虽然这个方案有些粗暴，但是它不涉及任何算法，更像是一个脑经急转弯，实现成本相当低。针对性能问题，如果可以迁移到 WebGL 上会有不小的提升（嗯？门槛好像变高了？）,  [pixi.js 的描边滤镜](https://github.com/pixijs/pixi-filters/tree/master/filters/outline) 就是采用这个方案。

## 轮廓提取

为什么 Canvas 内置了文字描边呢？因为文字已经自带了路径，所以直接绘制路径就完事了。那如果我们能够提取出图片的轮廓路径，是不是一切问题就迎刃而解了呢？

我们通过使用 [Marching squares 算法](https://en.wikipedia.org/wiki/Marching_squares) 能够从图像中提取出轮廓，得到轮廓路径后，之后只需要将路径绘制出来就行了。为了达到描边边缘圆润的效果，我们需要设置 `lineJoin` 为 `round`.

```javascript
const outlineWidth = 20
const path = getPath(image)
ctx.lineJoin = 'round'
ctx.lineWidth = outlineWidth * 2
drawPath(ctx, path)
ctx.drawImage(image)
```

再来看看结果，就算是大半径的描边也能正常输出：

![outline-by-marching-squares](https://user-gold-cdn.xitu.io/2020/3/10/170c0287c6f65af5?w=1382&h=1364&f=png&s=258413)

这个方案好像又快又好，而且也能处理描边宽度过大的情况。不过还是勉强能挑出缺点：

- 描边边缘还是不够平滑，如下：

![outline-by-marching-squares-edge](https://user-gold-cdn.xitu.io/2020/3/10/170c028e487f0576?w=1436&h=1432&f=png&s=1309071)

- 路径越多，绘制就需要越长时间。对此，可以通过一些 [路径简化算法](https://en.wikipedia.org/wiki/Ramer%E2%80%93Douglas%E2%80%93Peucker_algorithm) 来减少路径点。

## Distance transform

在轮廓提取的方向上还有另一个思路，我们能够得到图像的边缘之后，再算出整张图像里每个像素点到最近的边缘的距离。当描边宽度等于这个距离时，我们就填充这个像素点，这样便实现了描边。

[Distance transform](https://en.wikipedia.org/wiki/Distance_transform) 是一种计算二值图各像素点到边缘距离的算法。通过一段简单的代码理解一下这个算法：

``` javascript
const getPixelByPosition = (pixels, x, y) => { alpha: 0 }
const checkTransparent = pixel => pixel.alpha < 255
// 欧拉距离计算
const euclideanDistance = (x1, y1, x2, y2) => (
  sqrt(Math.pow(x1 - x2, 2) + Math.pow(y1 - y2, 2))
)

for(let pixel of pixels) {
  const isTransparent = checkTransparent(pixel)
  let { x, y } = pixel
  let min = Infinity
  // 判断目标是否位于图像边缘
  for(let ox = 0; ox < width; ox++) {
    for(let oy = 0; oy < height; oy++) {
        const current = getPixelByPosition(pixels, ox, oy)
        if(
          // 当前像素透明并且目标像素不透明
          isTransparent && !checkTransparent(current) ||
          // 当前像素不透明并且目标像素透明
          !isTransparent && checkTransparent(current)
        ) {
          min = Math.min(euclideanDistance(x, y, ox, oy), min)
        }
    }
  }
}
```

一句话说明就是逐像素地查找距离边缘的最短距离。不过这段代码复杂度太高了，实际场景根本无法用。我们可以选择现成的优秀算法，不过无论如何优化，复杂度也低不了多少，经过测试，2000 * 2000px 的图像需要 300ms。所以对于大尺寸图像，这个方案注定快不起来。尽管如此，当只要计算出距离数据后，之后的渲染和更新都不再是问题，我们可以轻松得做到实时更新描边结果。

另外，这类像素操作如果不经过抗锯齿的处理往往会产生「毛刺」，实时 CPU 锯齿计算显然不是一个好选择，于是我们就只剩 WebGL 可用了。那么在 WebGL 中如何解决这类简单的「毛刺」呢？在 [The Book Of Shaders](https://thebookofshaders.com/07/) 中通过 `smoothstep` 画出了一个更 「圆」 的圆，我们也可以基于此函数来解决这个「毛刺」问题。

这个方案除了初始化距离数据的时间过长以外，几乎没有其他缺点，并且相比其他方案，我们可以通过使用 [不同的距离函数](http://parmanoir.com/distance/) 来达到不同的描边效果。这个方案有不少现成的应用，例如 [tiny-sdf](https://mapbox.github.io/tiny-sdf/)。

## 总结

看似简单的描边，却有着不简单的方案。总结一下以上三个方案，这几个方案都各有优缺点，从性能、效果和门槛三个维度上来看排名大致是如下（针对 2000 * 2000px 的图像而言）：

- 性能：轮廓提取 > 图像偏移 > Distance Transform（初始久）
- 效果：Distance Transform >= 图像偏移 > 轮廓提取
- 门槛：Distance Transform > 轮廓提取 > 图像偏移

最后，放上基于本文的 [实践仓库](https://github.com/liajoy/image-stroke/) 与 [在线预览](https://liajoy.github.io/image-stroke/example-dist/)。

## 参考

- [SVG Filter Effects: Outline Text with <feMorphology>](https://tympanus.net/codrops/2019/01/22/svg-filter-effects-outline-text-with-femorphology/)
- [Marching squares](https://en.wikipedia.org/wiki/Marching_squares)
- [Ramer–Douglas–Peucker algorithm](https://en.wikipedia.org/wiki/Ramer%E2%80%93Douglas%E2%80%93Peucker_algorithm)
- [Distance transform](https://en.wikipedia.org/wiki/Distance_transform)
- [Implementation of the Distance Transform on OpenCL (CUDA, GPU)](http://www.david-web.appspot.com/cnt/OpenCLDistanceTransform/)
- [The Book of Shaders: Shapes](https://thebookofshaders.com/07/)
- [Meijster distance](http://parmanoir.com/distance/)

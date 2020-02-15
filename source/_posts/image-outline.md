---
title: WEB 中的透明图像描边
date: 2019-12-23 00:11:14
tags:
- 图像处理
- WebGL
- Canvas
---
图像描边是设计软件中常见的图像处理功能，在 Photoshop 中，图像应用的描边后的效果是这样：

![outline-in-ps](/images/outline-in-ps.png)

那么如何在 WEB 中实现这样的一个描边呢？笔者在 Google 上混迹了有一段时间，发现了这个功能并不简单，本文是记录关于 WEB 中描边的一些常见实现。

## SVG 滤镜

SVG 里有许多有趣的滤镜，其中的 `feMorphology` 可以达到将某些元素进行「扩张」或者「腐蚀」的效果。我们可以将它用于 [文字描边](https://tympanus.net/codrops/2019/01/22/svg-filter-effects-outline-text-with-femorphology/)。那如果将它应用在图像上呢？

![outline-by-svg-filter](/images/outline-by-svg-filter.png)

[demo](https://codepen.io/liajoy/pen/KKwmwom)

好吧，看起来效果和我们的预期相去甚远，这条路基本走不下去了。

## 图像偏移

我们先选择一张简单的矩形图像，如果将它进行填充并复制 8 份，把这 8 张分别沿着上、下、左、右、左上、左下、右上、右下八个方向进行偏移，就能完成对矩形图像的描边。不过它的描边结果不「圆润」，那么如果复制 360 份，让图像往 360 个方向进行偏移不就能做出圆角了吗？

但是这个方案有三个缺点:

  1. 耗时长，以一张 2000 * 2000px 的图像为例，在 Chrome 下完成一次描边需要 150ms 左右，而在 firefox 下需要 1s ，这也就意味着我们可能无法实时应用描边。
  2. 当描边的宽度超过了实际的图像尺寸后会出现镂空的现象，所以在描边宽度有限制。
  ![outline-by-offset-bad-result](/images/outline-by-offset-bad-result.png)
  3. 无法实现内描边。

虽然这个方案有些粗暴且「残疾」，但是它没有任何依赖，实现成本相当低。针对性能问题，如果可以迁移到 WebGL 上会有不小的提升。

## 轮廓提取

仔细想想，描边说到底不就是描出边缘吗？如果能够提取出图像的边缘，是不是一切问题就迎刃而解了呢？

我们通过使用 [Marching squares 算法](https://en.wikipedia.org/wiki/Marching_squares) 能够从图像中提取出轮廓，得到轮廓路径后，之后只需要将路径绘制出来就行了。为了达到描边边缘圆润的效果，我们需要设置 `lineJoin` 为 `round`.

```javascript
const outlineWidth = 20
const path = getPath(image)
ctx.lineJoin = 'round'
ctx.lineWidth = outlineWidth * 2
drawPath(ctx, path)
ctx.drawImage(image)
```

再来看看结果：

![outline-by-marching-squares](/images/outline-by-marching-squares.png)

这个方案好像又快又好，而且也能处理描边宽度过大的情况。不过还是有一些缺点：

- 仔细观察，描边边缘还是不够平滑，如下：

![outline-by-marching-squares-edge](/images/outline-by-marching-squares-edge.png)

- 路径越多，绘制就需要越长时间，可以通过一些 [路径简化算法](https://en.wikipedia.org/wiki/Ramer%E2%80%93Douglas%E2%80%93Peucker_algorithm) 来减少路径点。

## Distance transform

在轮廓提取的方向上还有另一个思路，当我们能够得到图像的边缘，再算出整张图像里每个像素点到最近的边缘的距离，当描边宽度等于这个距离时，我们就填充这个像素点，就此完成了描边。

[Distance transform](https://en.wikipedia.org/wiki/Distance_transform) 是一种计算二值图各像素点到边缘距离的算法。一段用于理解 distance transform 的代码如下（千万别跑这段代码）：

``` javascript
const getPixelByPosition = (pixels, x, y) => 'none'
const checkTransparent = pixel => pixel.alpha < 255
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

可以看到基本就是逐像素地查找最短距离，不过这个复杂度高到实际场景根本无法运行。好在有不少文章介绍了这个算法的优化，也有不少现成的实现。不过无论如何优化，计算的复杂度就摆在这，2000 * 2000px 的图像需要 300ms，所以在大尺寸图像的场景下这个方案注定快不起来。不过得到了距离数据后，之后的渲染和更新都不再是问题，我们可以轻松得做到实时更新描边结果。

同时，这类像素操作如果不经过抗锯齿的处理往往会产生「毛刺」，再在 CPU 上进行一些抗锯齿计算显然是不符合实际的，所以 WebGL 自然是唯一的选择。那么怎么如何解决「毛刺」呢？在 [这篇文章](https://thebookofshaders.com/07/) 中提到了如何画一个更 “圆” 的圆，通过文章里提到的 `smoothstep` 函数可以帮助我们绘制一个平滑的边缘。

这个方案除了计算距离数据所需的时间过长以外，几乎没有其他缺点，并且相比其他方案，我们可以通过使用 [不同的距离函数](http://parmanoir.com/distance/) 来达到不同的描边效果。这个方案有了一些现成的应用，例如 [tiny-sdf](https://mapbox.github.io/tiny-sdf/)。

## 总结

一个 “小小” 描边的坑越挖越深，越挖门槛越高，我已是无力再接着调研了。还是总结一下以上三个方案，这几个方案都各有优缺点，从性能、效果和门槛三个维度上来看排名大致是如下（针对 2000 * 2000px 的图像而言）：

- 性能：轮廓提取 > 图像偏移 > Distance Transform
- 效果：Distance Transform >= 图像偏移 > 轮廓提取
- 门槛：Distance Transform > 轮廓提取 > 图像偏移

## 参考

- [SVG Filter Effects: Outline Text with <feMorphology>](https://tympanus.net/codrops/2019/01/22/svg-filter-effects-outline-text-with-femorphology/)
- [Marching squares](https://en.wikipedia.org/wiki/Marching_squares)
- [Ramer–Douglas–Peucker algorithm](https://en.wikipedia.org/wiki/Ramer%E2%80%93Douglas%E2%80%93Peucker_algorithm)
- [Distance transform](https://en.wikipedia.org/wiki/Distance_transform)
- [Implementation of the Distance Transform on OpenCL (CUDA, GPU)](http://www.david-web.appspot.com/cnt/OpenCLDistanceTransform/)
- [The Book of Shaders: Shapes](https://thebookofshaders.com/07/)
- [Meijster distance](http://parmanoir.com/distance/)

---
title: 从字体文件到 3D 文字
date: 2019-03-24 00:08:43
tags:
- WebGL
---
## 字体类型

字体类型主要有：[点阵字体、轮廓字体、笔画字体](https://zh.wikipedia.org/wiki/%E8%AE%A1%E7%AE%97%E6%9C%BA%E5%AD%97%E4%BD%93)。根据描述方式的不同，可以分为点阵字体与矢量字体，它们的区别可以参考位图与矢量图。本文主要讨论轮廓字体中常见的 TrueType 标准（.TTF）下的相关实践。

![truetype-bitmap](/images//truetype-bitmap.gif)

## 字体解析

既然它是矢量字体，那么把字体当成 svg 看待不就简单了吗？首先对字体文件进行解析，这里使用 [typr.js](https://github.com/photopea/Typr.js)，解析过后，可以拿到字形的路径以及 metrics 信息。

![typr-result](/images//typr-result.png)

现在我们拿着路径可以在 canvas 2D 中进行绘制。

``` javascript
const path = new Path2D(pathStr);
canvas.stroke(path);
```

## triangulation

但是只有路径还不足以绘制 3D 文字。我们还需要对其进行三角化，将其分成很多个三角形，这些三角形将会是文字平面的组成部分。

![triangulation](/images//triangulation.png)

为了方便，以字母 i 为例，字母 i 共有 4 个路径点：1、2、3、4。

![i-path](/images//i-path.jpeg);

使用 [earcut](https://github.com/mapbox/earcut) 进行三角化转换。

``` javscript
var triangles = earcut([point1, point2, point3, point4]); // returns [1,0,3, 3,2,1]
```

这样就得到了两个三角形三个顶点对应的索引。

## 3D 化

经过了三角化，我们已经可以得到文字的一个平面，现在可以准备将它从 2D 转换成 3D 了。

如果将前面得到的那个平面作为正面，那么其他面要如何得到呢？

3D 化的字母 i 可以看作近似立方体，一个立方体需要 8 个顶点，而我们现在只有 4 个顶点。如果将得到的 4 个顶点组成的面作为正面，那么可以将这 4 个顶点沿 z 轴偏移，这样就能够拿到新的 4 个顶点作为背面。最后得到的顶点数据就是以下八个点：

``` javascript
[0,1,2,3,4,5,6,7]
```

![cube-index](/images//cube-index.jpg)

要得到背面，我们需要将正面的索引反转，反转之后加上总的顶点数：

```
[0, 1, 2] -> [0, 1, 2].reverse().map(i = i + verticesNum)
```

侧面复杂一些，把正面的 4 个顶点分成 4 段（每两个相邻的点），将它们与背面上对应的点连接就能得到一个面，而这个面就是我们需要的侧面。例如，0、1、5、4 组成的面，我们可以将它拆分成 `[0, 1, 5]` 与 `[5, 4, 0]`。

``` javascript
// [0, 1, 5, 4] -> [0, 1, 5] && [5, 4, 0]

for(let i = 0; i < verticesNum; i++) {
    if(i === verticesNum - 1) {
        const a = i;
        const b = i + 4;
        const c = i + 3;
        const d = i + 7;
    }

    const a = i;
    const b = i + 1;
    const c = i + 5;
    const d = i + 4;

    console.log(
        [a, b, c],
        [c, d, a]
    );
}

```

一通操作后，拿着得到的顶点数据和索引（面）丢进随便一个 [webgl demo](https://github.com/mdn/webgl-examples/tree/gh-pages/tutorial/sample5) 就搞定了。

``` javascript
gl.bindBuffer(gl.ELEMENT_ARRAY_BUFFER, faces);
gl.drawElements(gl.TRIANGLES, 4 * faces.length, gl.UNSIGNED_SHORT, 0);
```

## 参考资料

[文字渲染的那些事（一）字体是如何存储的？](https://juejin.im/post/5c1e2e576fb9a049d5197af2)

[文字渲染那些事（二）文字的形状是怎么表示的？](https://juejin.im/post/5c54f19b6fb9a049d05e2b57)

[three source](https://github.com/mrdoob/three.js/)
---
title: math-of-cropper
date: 2020-01-07 23:33:50
tags:
- 幼儿数学
---
一年半前，开发了一个图片编辑器，里边的裁剪功能折磨的我死去活来。最近，裁剪这个需求又被搬了出来。再回头看那么久远的代码，果然全都忘了。就此重新找一下思路。

抛开冗长的需求描述，归结到的第一个数学问题就是：下图中黑色白色矩形的两个顶点位于黑色矩形的边上，黑色矩形的宽高比例与白色矩形的宽高比例相等，已知白色矩形的宽高和与黑色矩形的旋转角度，求黑色矩形的宽高。

![two-rect](../images/two-rect.png)

这个问题一下把我带回了 2008 年，申奥成功都这么多年了，没想到我还得做几何题。不过仅凭着残余的一些数学知识，折腾了一会还是能勉强解出来的。

![rect-analyze](../images/rect-analyze.png)

步骤如下：

1. 已知内部矩形的宽高，可以求出角 B 的。
2. 已知角 A 为旋转的角度。
3. 因为角 C 与角 A 为互补角，所以角 C 等于 角 A。
4. 作垂直于外部矩形一边的辅助线。
5. 已知角 B 与角 C 以及内部矩形的斜边，所以可以求出辅助线的长度。

代码呢，就是这样：

``` javascript
const getOutRectSize = (innerRect, rotation) => {
    const hypotenuse = Math.hypot(innerRect.width, innerRect.height);
    const angleB = Math.asin(innerRect.height / hypotenuse) * 180 / Math.PI;
    const angleC = rotation;
    const angleSum = angleB + angleC;
    const height = Math.sin(angleSum / 180 * Math.PI) * hypotenuse;
    return {
        width: innerRect.width * (height / innerRect.height),
        height
    }
}

```

2020 年了，希望以后做需求不做数学题，再简单也不行。

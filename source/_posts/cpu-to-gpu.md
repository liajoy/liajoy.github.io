---
title: 从 CPU 到 GPU
date: 2019-05-22 23:57:27
tags:
- 计算机基础
---
## 从 CPU 出发

- 收集需要渲染的 mesh 信息与资源。（hdd -> ram)
- CPU 将包含数据（shader、texture、material、lighting、transparency 等）的 setPass call 命令并将其放进 commandBuffer。
- CPU 将包含数据（渲染对象）的 draw call 放进 commandBuffer 并发送给 GPU。

## 到达 GPU

- GPU 收到并按照顺序读取 commandBuffer。
- GPU 根据 set render state 命令更新 render state。
  - Gigathread Engine 从 ram 中获取信息并复制到 vram。
- GPU 根据 draw call 命令中的数据按照当前 render state 进行绘制。
  - 将每个顶点、像素及相关数据打包成 thread block 分发给 SM 单元。
  - SM 单元中的 Polymorph Engine 将顶点数据从 vram 复制到 l1、l2、register。
  - SM 单元将 thread block 拆分成更小的 warp 用于调度并发出命令交予 core 执行，接下来就是 pipeline 流程，其中有 core 与 GPU 上的其他单元参与。core 在其中的角色就是对着色器的每一行代码进行执行。

![SM](/images//SM.jpg)

### 渲染管线

![pipeline](/images//pipeline.png)

![pipeline-output](/images//pipeline-output.jpg)

- Vertex Processing：执行 Vertex shader。
- Primitive Assembly：根据图元类型将 Vertex 组合为图元并输出。
- Tessellation Shading：曲面细分。
- Geometry Shading：新增、更改、删除图元。
- Viewport Transform & Clipping & ：视口变换，裁剪超出视口的图元。
- Culling & Rasterization：将图元转为输出 fragments，以及 sample coverage（用作 MSAA ） 信息。
- Fragment Processing：执行 Fragment Shader。
- Raster Output：
  - frameBuffer blend
  - depth test
  - antiAliasing

## 参考

- [Optimizing graphics rendering in Unity games](https://unity3d.com/learn/tutorials/temas/performance-optimization/optimizing-graphics-rendering-unity-games)
- [A Modern Multi-Core Processor](http://15418.courses.cs.cmu.edu/spring2015/lecture/basicarch/)
- [计算机那些事(8)——图形图像渲染原理](http://chuquan.me/2018/08/26/graphics-rending-principle-gpu/)
- [Render Hell](https://simonschreibt.de/gat/renderhell-book2/)
- [How GPU Work](https://www.cs.cmu.edu/afs/cs/academic/class/15462-f11/www/lec_slides/lec19.pdf)
- [Life of a triangle - NVIDIA's logical pipeline](https://developer.nvidia.com/content/life-triangle-nvidias-logical-pipeline)

---
layout:     post
title:      "《OpenGL ES 应用开发实践指南》读书笔记 No.2"
subtitle:   "Android OpenGL ES 从入门到奔溃"
date: 2016-07-18 11:00:01 +0800
author:     "Roger"
header-img: "img/opengl-bg.jpg"
tags:
    - OpenGL ES
---
Android OpenGL ES 第二章 - 顶点和着色器
---

本系列所有源码地址：[https://github.com/Rogero0o/OpenGL_Demo](https://github.com/Rogero0o/OpenGL_Demo)

上一章我们学习了如何搭建最基本的 OpenGL Android 程序，接下来我们学习顶点和着色器

### 顶点

在 OpenGL 中，所有的物体都是由顶点集合来构建的，一个顶点就是一个代表几何对象的拐角，有很多附加属性，最重要的属性就是位置，代表了这个顶点在空间中的定位。

这一章我们要学习用顶点来构建一个桌子，将之前的项目复制一份，重命名为 AirHockey1。

我们需要构建的桌子顶点如图：

![image](https://github.com/Rogero0o/rogero0o.github.io/blob/master/img/opengl/image2-2.jpg?raw=true)

如何保存这些顶点呢？这些顶点辉表示为一个浮点数列表，一个标记 x 轴位置，一个标记 y 轴位置。在 AirHockeyRenderer 类中，加入以下常量：

    private static final int POSITION_COMPONENT_COUNT = 2;

    float[] tableVertices = {
          0f,  0f,
          0f, 14f,
          9f, 14f,
          9f,  0f
    };

这四个点为桌面的四个顶点，然而需要注意的是，**在 OpenGL 中，只能绘制点、直线以及三角形！**

所以我们要将一个正方形切割为两个三角形：

![image](https://github.com/Rogero0o/rogero0o.github.io/blob/master/img/opengl/image2-3.jpg?raw=true)

因此，需要将 tableVertices改为：

    float[] tableVerticesWithTriangles = {
      // Triangle 1
      0f,  0f,
      9f, 14f,
      0f, 14f,

      // Triangle 2
      0f,  0f,
      9f,  0f,							
      9f, 14f
    }

还有一个需要注意的点是，**在定义三角形时，我们总是以逆时针的顺序排列顶点**，这称为卷曲顺序。以为在任何地方都使用这种一致的卷曲顺序排列顶点，可以优化性能，可以指出一个三角形输入任何给定物体的前面或者后面， OpenGL 可以忽略那些无论如何都无法被看到的后面的三角形。

完成这些之后，我们已经在 java 层定义好了顶点，然而这个定义并不能被 OpenGL 直接使用，我们需要将这些数据复制到本地堆：

![image](https://github.com/Rogero0o/rogero0o.github.io/blob/master/img/opengl/image2-4.jpg?raw=true)

在构造函数的结尾处加入以下代码：

    vertexData = ByteBuffer
                .allocateDirect(tableVerticesWithTriangles.length * BYTES_PER_FLOAT)
                .order(ByteOrder.nativeOrder())
                .asFloatBuffer();

    vertexData.put(tableVerticesWithTriangles);

*ByteBuffer.allocateDirect* 方法分配了一块本地内存，这块内存是不会被垃圾回收机制回收的，参数是分配多少字节的内存块。 *order(ByteOrder.nativeOrder())* 告诉字节缓冲区按照本地字节序组织它的内容。最后，调用 *asFloatBuffer()* 得到一个可以反映底层字节的 FloatBuffer 类实例。再调用 *vertexData.put(tableVerticesWithTriangles);* 将数据从虚拟机复制到本地内存。

上面有很多专有名词的确是很难懂，但是我们不必纠结于细节，只需要大概了解整个方法的功能和用途即可。

### 着色器

接下来是着色器部分，这部分是比较难懂的，有说得不对的地方请大家拍砖指正。

任何 OpenGL 程序都需要两种着色器，顶点着色器 ( vertex shader ) 和片段着色器 ( fragment shader ) 。

1. 顶点着色器生成每个顶点的最终位置，针对每个顶点，它都会执行一次，一旦最终确定了位置，OpenGL 就可以根据这些顶点的集合组装成点、直线以及三角形。
2. 片段着色器为组成点、线、三角形的每个片段生成最终的颜色，针对每个片段，它都会执行一次，一个片段是一个小的、单一颜色的长方形区域，类似于计算机屏幕上的一个像素。一旦最终颜色生成了，OpenGL 就会把它们写到一块称为帧缓冲区的内存块中，然后 Android 会把这个块显示到屏幕上。

这个过程可以简单的归纳为七个阶段：读取顶点数据 -> 执行顶点着色器 -> 组装图元 -> 光栅化图元 -> 执行片段着色器 -> 写入帧缓冲区 -> 显示在屏幕上 。

#### 创建一个顶点着色器

在 res 的 raw 文件夹中创建一个顶点着色器 simple_vertex_shader.glsl 。

    attribute vec4 a_Position;

    void main()
    {
      gl_Position = a_Position;
      gl_PointSize = 10.0;
    }

*attribute* 是一种 OpenGL 中的类型修饰符，建议到 [Link](http://blog.csdn.net/renai2008/article/details/7844495) 中了解一下详细。*vec4* 是包含四个分量的向量，如果没有指定，默认情况下前三个分量都为0，最后一个为1。之后通过 *gl_Position* 将位置信息传给 OpenGL ，顶点着色器必须对 *gl_Position* 赋值。 OpenGL 会把这个值作为当前顶点的最终位置，并把这些顶点组装成点、直线和三角形。

### 创建一个片段着色器

在此之前，你需要知道什么是光栅化：[Link](https://www.zhihu.com/question/29163054)

接下来让我们创建一个片段着色器，在 res 的 raw 文件夹中创建一个片段着色器 simple_fragment_shader.glsl 。

    precision mediump float;

    uniform vec4 u_Color;

    void main()
    {
      gl_FragColor = u_Color;
    }

*precision* 是默认精度修饰符，具体可参考 [Link](http://blog.csdn.net/hgl868/article/details/7846269) 4.5.3 中的内容。 mediump 表示使用中等精度的值，就像java中选择是使用 float 还是 double 类似。 *uniform* 在 [Link](http://blog.csdn.net/renai2008/article/details/7844495) 中有解释，在片段着色器中，我们需要对 *gl_FragColor* 进行赋值， OpenGL 会使用这个颜色作为当前片段的最终颜色。

这章我们学习了顶点和着色器，这些都是 OpenGL 中的重点难点，后边课程中都是基于顶点和着色器的扩展和细化，所以一些基础知识还是需要反复记忆巩固的。下一章我们将学习如何编译着色器以及在屏幕上显示。

[《OpenGL ES 应用开发实践指南》读书笔记 No.1](http://www.rogerblog.cn/2016/07/18/OpenGL-serise-No1/)

[《OpenGL ES 应用开发实践指南》读书笔记 No.2](http://www.rogerblog.cn/2016/07/18/OpenGL-serise-No2/)

[《OpenGL ES 应用开发实践指南》读书笔记 No.3](http://www.rogerblog.cn/2016/07/19/OpenGL-serise-No3/)

[《OpenGL ES 应用开发实践指南》读书笔记 No.4](http://www.rogerblog.cn/2016/07/20/OpenGL-serise-No4/)

[《OpenGL ES 应用开发实践指南》读书笔记 No.5](http://www.rogerblog.cn/2016/07/20/OpenGL-serise-No5/)

[《OpenGL ES 应用开发实践指南》读书笔记 No.6](http://www.rogerblog.cn/2016/07/21/OpenGL-serise-No6/)

[《OpenGL ES 应用开发实践指南》读书笔记 No.7](http://www.rogerblog.cn/2016/07/22/OpenGL-serise-No7/)

[《OpenGL ES 应用开发实践指南》读书笔记 No.8](http://www.rogerblog.cn/2016/07/24/OpenGL-serise-No8/)

[《OpenGL ES 应用开发实践指南》读书笔记 No.9](http://www.rogerblog.cn/2016/07/26/OpenGL-serise-No9/)

---
layout:     post
title:      "《OpenGL ES 应用开发实践指南》读书笔记 No.6"
subtitle:   "Android OpenGL ES 从入门到奔溃"
date: 2016-07-21 11:00:01 +0800
author:     "Roger"
header-img: "img/opengl-bg.jpg"
tags:
    - OpenGL ES
---
Android OpenGL ES 第六章 - 进入三维世界
---

本系列所有源码地址：[https://github.com/Rogero0o/OpenGL_Demo](https://github.com/Rogero0o/OpenGL_Demo)

请大家务必对照源码阅读本文，否则有如盲人摸象。

上一章我们学习了如何调整屏幕宽高比，这一章我们将学习如何进入三维世界。这一章的项目名为 AirHockey3D 。

### 三维的艺术

在本章，我们会学习关于 OpenGL 的透视除法的内容，以及如何在二维屏幕上使用 w 分量创造三维的假象，一旦理解了 w 分量，我们就会学习如何设置透视投影，让我们可以见到三维形式的桌子。

让我们先来学习一下从着色器到屏幕的坐标变换，如下图所示：

![image](https://github.com/Rogero0o/rogero0o.github.io/blob/master/img/opengl/image6-1.jpg?raw=true)

### 透视除法

如上图，透视除法之后，那个位置就在归一化设备坐标中了，不管渲染区域的大小和形状，对于其中的每个可视坐标，其 x,y,z 分量的取值都在 [-1,1] 的范围内。为了在屏幕上创建三维的幻想， OpenGL 会把每个 gl_Position 的 x,y 和 z 分量都除以它的 w 分量。当 w 分量用来表示距离的时候，就使得比较远的物体被移动到距离渲染区域中心更接近的地方，这个中心的作用就像一个消失点。如图所示，w 分量类似于距离的概念，数值越大表示离得越远：

![image](https://github.com/Rogero0o/rogero0o.github.io/blob/master/img/opengl/image6-2.jpg?raw=true)

如果不是很通透，或许需要上网找一下相关知识，例如 [Link](http://blog.csdn.net/luyuncsd123/article/details/12955883) 这里的第 10 点中的说明可能会比较详细。下一步我们将添加 w 分量创建三维视图，在 AirHockeyRenderer.java 中做如下修改：

    private static final int BYTES_PER_FLOAT = 4;

    float[] tableVerticesWithTriangles = {   
            // Order of coordinates: X, Y, R, G, B

            // Triangle Fan
               0f,    0f,   1f,   1f,   1f,         
            -0.5f, -0.5f, 0.7f, 0.7f, 0.7f,            
             0.5f, -0.5f, 0.7f, 0.7f, 0.7f,
             0.5f,  0.5f, 0.7f, 0.7f, 0.7f,
            -0.5f,  0.5f, 0.7f, 0.7f, 0.7f,
            -0.5f, -0.5f, 0.7f, 0.7f, 0.7f,

            // Line 1
            -0.5f, 0f, 1f, 0f, 0f,
             0.5f, 0f, 1f, 0f, 0f,

            // Mallets
            0f, -0.25f, 0f, 0f, 1f,
            0f,  0.25f, 1f, 0f, 0f
    };

我们给顶点加入了一个 z 和 w 分量，其中接近底部的顶点 w 为 1 ，接近顶部的顶点 w 设为2，运行这个项目，你将发现现在看起来好像是一个三维的桌子了。通过这个例子我们了解到如何使用 w 分量来完成三维视图，然而这是通过硬编码的方法，接下来，我们将学习如何使用透视投影矩阵来自动生成 w 的值。

我们将使用 z 分量来作为物体与焦点的距离并且把这个距离映射到 w ，这个距离越大，w 就越大，所得到的物体就越小。接下来我们在代表中创建投影矩阵。在 util 包中的 MatrixHelper 中，我们看到如下方法：

    public static void perspectiveM(float[] m, float yFovInDegrees, float aspect,
        float n, float f) {
        final float angleInRadians = (float) (yFovInDegrees * Math.PI / 180.0);

        final float a = (float) (1.0 / Math.tan(angleInRadians / 2.0));
        m[0] = a / aspect;
        m[1] = 0f;
        m[2] = 0f;
        m[3] = 0f;

        m[4] = 0f;
        m[5] = a;
        m[6] = 0f;
        m[7] = 0f;

        m[8] = 0f;
        m[9] = 0f;
        m[10] = -((f + n) / (f - n));
        m[11] = -1f;

        m[12] = 0f;
        m[13] = 0f;
        m[14] = -((2f * f * n) / (f - n));
        m[15] = 0f;        
    }

这个方法完成了透视投影的功能。接下来我们将使用它，在 AirHockeyRenderer 的 onSurfaceChanged() 方法做如下改变：

    glViewport(0, 0, width, height);
    MatrixHelper.perspectiveM(projectionMatrix, 45, (float) width / (float) height, 1f, 10f);

这会用 45 度的视野创建一个透视投影。这个视锥体从 z 值为-1的位置开始，在z值为-10的位置结束。但是这时运行程序我们是无法看到任何东西的！因为我们没有给桌子定义 z 的位置，默认情况下它们处于 z 的位置，因为这个视锥体是从 -1 开始的，所以我们无法看到桌子，除非把它移到那个距离内。关于视锥体以及投影矩阵的知识可以参考 [Link](http://www.cnblogs.com/graphics/archive/2012/07/25/2582119.html)

接下来我们将利用模型矩阵移动物体，在 AirHockeyRenderer 中添加如下声明：

    private final float[] modelMatrix = new float[16];

在 onSurfaceChanged() 的结尾处，加入如下代码：

    translateM(modelMatrix, 0, 0f, 0f, -2.5f);        

这样就完成了对桌子在 z 轴上往负方向移动了2个单位。

现在我们完成了移动桌子的步骤，让我们想一下流程，我们需要先将每一个顶点乘以这个移动矩阵，完成在 z 轴上的移动，然后再乘以投影矩阵，这样才完成了所有工作。那么能否将这个移动矩阵和投影矩阵相乘呢？答案是可以的。在代码中体现如下，在 translateM() 方法后加入如下代码：

    final float[] temp = new float[16];
    multiplyMM(temp, 0, projectionMatrix, 0, modelMatrix, 0);        
    System.arraycopy(temp, 0, projectionMatrix, 0, temp.length);

*multiplyMM()* 这个方法将模型矩阵（移动矩阵），和投影矩阵相乘得到一个最终使用的矩阵 temp ，最后使用 *System.arraycopy()* 把结果存回 *projectionMatrix* 中。

### 增加旋转

现在我们已经有了一个配置好的投影矩阵和一个可以移动桌子的模型矩阵，那么我们需要做的就是旋转这张桌子，以便可以从某个角度观察它。要实现这个效果，只需要一行代码就可以：

    rotateM(modelMatrix, 0, -60f, 1f, 0f, 0f);

关于 rotateM() 这个方法，参数介绍如下：

    rotateM(float[] m, int mOffset, float a, float x, float y, float z)
    Rotates matrix m in place by angle a (in degrees) around the axis (x, y, z).

把桌子绕着 x 轴旋转了 -60 度，完成最后的工作，现在这个桌子看起来就像一个三维的物体了。

这一章我们主要学习了如何用矩阵移动和旋转物体，对于这其中的矩阵原理和理论，我们暂时不需要深入的理解，只需要他们的功能即可。下一章我们将学习用纹理增加细节。

[《OpenGL ES 应用开发实践指南》读书笔记 No.1](http://www.rogerblog.cn/2016/07/18/OpenGL-serise-No1/)

[《OpenGL ES 应用开发实践指南》读书笔记 No.2](http://www.rogerblog.cn/2016/07/18/OpenGL-serise-No2/)

[《OpenGL ES 应用开发实践指南》读书笔记 No.3](http://www.rogerblog.cn/2016/07/19/OpenGL-serise-No3/)

[《OpenGL ES 应用开发实践指南》读书笔记 No.4](http://www.rogerblog.cn/2016/07/20/OpenGL-serise-No4/)

[《OpenGL ES 应用开发实践指南》读书笔记 No.5](http://www.rogerblog.cn/2016/07/20/OpenGL-serise-No5/)

[《OpenGL ES 应用开发实践指南》读书笔记 No.6](http://www.rogerblog.cn/2016/07/21/OpenGL-serise-No6/)

[《OpenGL ES 应用开发实践指南》读书笔记 No.7](http://www.rogerblog.cn/2016/07/22/OpenGL-serise-No7/)

[《OpenGL ES 应用开发实践指南》读书笔记 No.8](http://www.rogerblog.cn/2016/07/24/OpenGL-serise-No8/)

[《OpenGL ES 应用开发实践指南》读书笔记 No.9](http://www.rogerblog.cn/2016/07/26/OpenGL-serise-No9/)

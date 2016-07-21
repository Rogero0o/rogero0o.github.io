---
layout:     post
title:      "《OpenGL ES 应用开发实践指南》读书笔记 No.5"
subtitle:   "Android OpenGL ES 从入门到奔溃"
date: 2016-07-20 11:00:01 +0800
author:     "Roger"
header-img: "img/opengl-bg.jpg"
tags:
    - OpenGL ES
---
Android OpenGL ES 第五章 - 调整屏幕宽高比
---

本系列所有源码地址：[https://github.com/Rogero0o/OpenGL_Demo](https://github.com/Rogero0o/OpenGL_Demo)

请大家务必对照源码阅读本文，否则有如盲人摸象。

上一章我们学习了如何增加颜色和着色，这一章我们将学习调整屏幕宽高比。这一章的项目名为 AirHockeyOrtho 。

### 宽高比问题

上一章中我们已经完成了一个中间明亮四周暗淡的桌子，如果将程序运行起来后将会发现在横竖屏切换时桌子的比例将会变得很奇怪，所以这一章我们将解决宽高比问题。为了适应宽高比，我们需要调整坐标空间，使它把屏幕的形状考虑在内，可行的方法是把较小的范围固定在[-1,1],而按屏幕尺寸的比例调整较大的范围。举例来说，在竖屏情况下，宽是720，高是1280，因此我们可以把宽度范围限定在[-1,1]，高度范围调整为[-1280/720,1280/720]或是[-1.78,1.78]，同理，在横屏模式下，将高度范围设为[-1.78,1.78]，高度范围设为[-1,1]。如图所示：

![image](https://github.com/Rogero0o/rogero0o.github.io/blob/master/img/opengl/image5-1.jpg?raw=true)

接下来我们会使用正交投影，若忘了啥是正交投影请自行百度。

要使用正交投影，我们可以通过使用 Android 的 Matrix 类，其中有一个 *orthoM()* 的方法，它可以为我们生成一个正交投影。

### 加入正交投影

现在让我们更新代码加入正交投影。需要做的第一件事就是更新着色器，以便它用矩阵变换位置。打开项目的 simple_vertex_shader.glsl 并更新如下：

    uniform mat4 u_Matrix;

    attribute vec4 a_Position;
    attribute vec4 a_Color;

    varying vec4 v_Color;

    void main()
    {
      v_Color = a_Color;

      gl_Position = u_Matrix*a_Position;
      gl_PointSize = 10.0;
    }

我们添加了一个 u_Matrix ,并把它定位为一个 mat4 类型，意思是这个 uniform 代表了一个 4x4 的矩阵。 看到赋值这一行：

    gl_Position = u_Matrix*a_Position;

表示顶点着色器中的位置信息将被矩阵变换后才展示。

接下来我们在 java 代码中定义这个值，打开 AirHockeyRenderer ，添加如下定义：

    private static final String U_MATRIX = "u_Matrix";
    private final float[] projectionMatrix = new float[16];
    private int uMatrixLocation;

这里定义了着色器中那个新 uniform 的名字，定义了一个顶点数组用于存储那个矩阵，还定义了一个整形来存储矩阵 uniform 的位置。然后，我们需要再 onSurfaceCreated() 中加入如下代码：

    uMatrixLocation = GLES20.glGetUniformLocation(program,U_MATRIX);

下一步就是更新 *onSurfaceChanged()*。在 *glViewport()* 调用后面加入如下代码：

    final float aspectRatio = width > height ?(float)width/(float)height:(float)height/(float)width;

    if(width>height){
      Matrix.orthoM(projectionMatrix,0,-aspectRatio,aspectRatio,-1f,1f,-1f,1f);
    }else{
      Matrix.orthoM(projectionMatrix,0,-1f,1f,-aspectRatio,aspectRatio,-1f,1f);
    }

这段代码会创建一个正交投影矩阵，这个矩阵会把屏幕的当前方向计算在内。我们首先计算了宽高比，它使用宽高中的较大值除以较小值，所有不管横屏还是竖屏，这个值都是一样的。然后调用 *orthoM()* 方法。在横屏模式下，我们会扩展宽度的坐标，以达到在横竖屏上都显示正常的效果。

最后我们需要将刚刚定义的矩阵传递给着色器，在 onDrawFrame() 中，在 glClear() 调用之后加入如下代码：

    glUniformMatrix4fv(uMatrixLocation,1,false,projectionMatrix,0);

完成这些改动后，我们可以运行看看。如果没有错，你将会看到一个正方形的桌子显示在屏幕上。等等？正方形？我们要的桌子应该是长方形的嘛。为了将桌子变成长方形，我们需要对顶点数据做一些如下改动：

    float[] tableVertices = {
        //Order of coordinates:X,Y,R,G,B


        //Triangle Fan
        0f,0f,1f,1f,1f,
        -0.5f,-0.8f, 0.7f, 0.7f, 0.7f,
        0.5f,-0.8f, 0.7f, 0.7f, 0.7f,
        0.5f, 0.8f, 0.7f, 0.7f, 0.7f,
        -0.5f, 0.8f, 0.7f, 0.7f, 0.7f,
        -0.5f,-0.8f, 0.7f, 0.7f, 0.7f,

        -0.5f,0f,1f,0f,0f,
        0.5f,0f,1f,0f,0f,

        0f,-0.25f,0f,0f,1f,
        0f, 0.25f,1f,0f,0f
    };

现在，这个空气曲棍球的桌子看起来就对了，而且在横竖屏中都保持同样的形状。

这一章我们学习了如何定义正交投影矩阵，并使用它来修复横竖屏的问题。下一章我们将学习如何实现三维的空气曲棍球桌子。

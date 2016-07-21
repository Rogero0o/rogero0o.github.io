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

上一章我们学习了如何编译着色器并且在屏幕上将其显示出来，这一章我们将学习如何增加颜色和着色。这一章的项目名为 AirHockey2 。

### 平滑着色

上一章中我们已经知道，在 OpenGL 中只能画点、直线和三角形，并且所有的物体都以它们为基础构成。那么如何表现许多不同的颜色呢？我们能使用的一个方法是用上百万个小三角形，每个三角形用不同的颜色，如果三角形足够多，就能欺骗观察者。由于平滑着色是在顶点间完成的，这一章的目标是使我们上一章的桌子中间看起来明亮边缘看起来暗淡。如何才能使中间更亮了？由于中间并没有点，所以我们需要改变桌子的构成，由两个三角形变为四个三角形。

![image](https://github.com/Rogero0o/rogero0o.github.io/blob/master/img/opengl/image4-1.jpg?raw=true)

在 *AirHockeyRenderer* 中，将坐标修改为以下数据：

    // Triangle Fan
       0,     0,            
    -0.5f, -0.5f,             
     0.5f, -0.5f,
     0.5f,  0.5f,
    -0.5f,  0.5f,            
    -0.5f, -0.5f,

如果细心的话将会发现，我们将要绘制四个三角形，所以应该需要12个顶点，然而这里只有六个顶点，这是怎么回事？这是因为这里使用了三角形扇面的绘制方法，具体来说就是以一个中心点为起始，使用相邻的两个顶点创建第一个三角形，接下来的每个顶点都会创建一个三角形，围绕起始的中心点按扇形展开。为了闭合这个扇形，我们需要再最后重复第二个点。关于绘制三角形扇，可以参考这个解析 [Link](http://book.2cto.com/201412/48540.html)

在更改完数据后，需要再在 onDrawFrame 中更新一下代码，把第一个 glDrawArrays 更新为如下代码，表示绘制三角形扇。

    // Draw the table.        
    glDrawArrays(GL_TRIANGLE_FAN, 0, 6);

下一步我们在数据中添加颜色属性，更新后如下：

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

如上，我们给每个顶点增加了三个额外的数字，这些数字代表红色，绿色和蓝色，它们一起构成了那个对应顶点的颜色。

下一步是给着色器添加颜色属性。打开 simple_vertex_shader.glsl  并更新如下：

    attribute vec4 a_Position;  
    attribute vec4 a_Color;

    varying vec4 v_Color;

    void main()                    
    {                            
        v_Color = a_Color;

        gl_Position = a_Position;    
        gl_PointSize = 10.0;          
    }

我们加入了一个新的属性 a_Color，也加入了一个叫做 v_Color 的 varying 。关于 varying 可以参考 [Link](http://blog.csdn.net/renai2008/article/details/7844495) 。

在片段着色器中也需要更新，打开 simple_fragment_shader.glsl 并更新如下：

    precision mediump float; 				
    varying vec4 v_Color;      	   								

    void main()                    		
    {                              	
        gl_FragColor = v_Color;                                  		
    }

我们已经为数据增加了一个颜色属性，并且更新了顶点和片段着色器，让它们使用这个属性，下一步就是去掉使用 unifom 传递颜色的旧代码，并告诉 OpenGL 吧颜色作为一个顶点属性读入。

在 AirHockeyRenderer 的顶部加入如下常量：

    private static final String A_COLOR = "a_Color";
    private static final int COLOR_COMPONENT_COUNT = 3;
    private static final int STRIDE = (POSITION_COMOPNENT_COUNT+COLOR_COMPONENT_COUNT) * BYTES_PER_FLOAT;
    private int aColorLocation;

加入后，既可以去掉旧的常量和 u_Color 了。下一步是更新 *onSurfaceCreated()* ,去掉 u_Color 相关的代码，并加入如下代码：

    aColorLocation = GLES20.glGetAttribLocation(program,A_COLOR);

下一步是更新 *glVertexAttribPointer()* 方法的调用，让它加入哪个跨距：

    glVertexAttribPointer(aPostionLocation,POSITION_COMOPNENT_COUNT,GLES20.GL_FLOAT,false,STRIDE,vertexData);

接下来就可以加入代码将顶点数据与着色器中的 a_Color 关联起来，在 onSurfaceCreated 中的末尾加入如下代码：

    vertexData.position(POSITION_COMOPNENT_COUNT);
    GLES20.glVertexAttribPointer(aColorLocation,COLOR_COMPONENT_COUNT,GLES20.GL_FLOAT,false,STRIDE,vertexData);
    GLES20.glEnableVertexAttribArray(aColorLocation);

这三行代码比较重要，让我们单独分析一下：

1. 我们把 *vertexData* 的位置设为 *POSITION_COMOPNENT_COUNT* ，也就是2，这是因为我们要读入颜色属性时，我们要从第一个颜色属性读起，而不是位置属性，而在源数据中，颜色属性的位置是2.
2. 接下来，我们调用 *glVertexAttribPointer* 把颜色数据和着色器中的 a_Color 关联起来。 STRIDE 这个参数是跨距，这个值告诉 OpenGL 两个颜色属性之间的距离是多少，这样当位置属性和颜色属性连接着存储时，就不会错误的将位置属性当做颜色属性读进来。
3. 像之前一样，我们需要使能属性数组。

接下来我们需要更新 onDrawFrame() ，删除 glUniform4f() 调用，因为我们不需要他们了。既然我们将顶点数据和 a_Color 关联起来了，只需要调用 glDrawArrays() 即可，OpenGL 会自动从顶点数据中读入颜色属性。完成这些后，运行程序，你将得到一个中间明亮周围暗淡的桌面了。

这一章我们主要学习了给每个顶点增加颜色，方法是给顶点数据和顶点着色器增加了一个新的属性，并且告诉 OpenGL 如何使用跨距读入属性。下一章我们将学习如何处理横竖屏的问题。

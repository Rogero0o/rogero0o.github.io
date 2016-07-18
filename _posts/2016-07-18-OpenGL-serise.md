---
layout:     post
title:      "《OpenGL ES 应用开发实践指南》读书笔记 No.1"
subtitle:   "Android OpenGL ES 从入门到奔溃"
date: 2016-07-18 11:00:01 +0800
author:     "Roger"
header-img: "img/post-bg-2015.jpg"
tags:
    - OpenGL ES
---
Android OpenGL ES 第一章
---

转眼已经16年了，最近出现了两个比较新潮的技术和产品，一个是 VR ，另一个是直播，而这两者直接或是简介的都和 OpenGL 有关，特此拜读了 《OpenGL ES 应用开发实践指南》 一书学习一些基本概念和知识，初步认识 OpenGL ，拓展知识面。读书只能掌握 30% 的知识，通过写博记录，加深理解和应用，也抛砖引玉，希望可以帮到对 OpenGL 有兴趣的同学。

# 初始化 OpenGL

想造自行车，那么第一步就是先学会怎么把自行车骑了。我们先搭建一个最简单的 OpenGL 实例，然后一步步细化，了解其中的奥秘。

本系列所有源码地址：[https://github.com/Rogero0o/OpenGL_Demo](https://github.com/Rogero0o/OpenGL_Demo)

首先新建第一个项目，源码中名为 AirHockey1 ，意思是一个空气曲棍球游戏，这是书中的代码教学用代码示例.

### GLSurfaceView

GLSurfaceView是一个视图，继承至SurfaceView，它内嵌的surface专门负责OpenGL渲染，所以我们要用它来初始化OpenGL。

GLSurfaceView提供了下列特性：

1. 管理一个surface，这个surface就是一块特殊的内存，能直接排版到android的视图view上。
2. 管理一个EGL display，它能让opengl把内容渲染到上述的surface上。
3. 用户自定义渲染器(render)。
4. 让渲染器在独立的线程里运作，和UI线程分离。
5. 支持按需渲染(on-demand)和连续渲染(continuous)。
6. 一些可选工具，如调试。

打开第一个项目的 Activity ，添加两个成员变量：

    private GLSurfaceView glSurfaceView;
    private boolean rendererSet = false;

并在 onCreate 中使用如下代码初始化 GLSurfaceView 。

    glSurfaceView = new GLSurfaceView(this);

如果你使用的是模拟器，请使用以下代码来判断是否支持 OpenGL ES 2.0

    final ActivityManager activityManager =
        (ActivityManager) getSystemService(Context.ACTIVITY_SERVICE);
    final ConfigurationInfo configurationInfo = activityManager.getDeviceConfigurationInfo();

    final boolean supportsEs2 = configurationInfo.reqGlEsVersion >= 0x20000;
    if (supportsEs2) {
      glSurfaceView.setEGLContextClientVersion(2);
      glSurfaceView.setRenderer(new AirHockeyRenderer(this));
      rendererSet = true;
    } else {
      Toast.makeText(this, "Not support OpenGL2.0", Toast.LENGTH_SHORT).show();
      return;
    }

最后调用

    setContentView(glSurfaceView);

将 GLSurfaceView 显示出来。

再处理一下生命周期事件

    @Override protected void onResume() {
      super.onResume();
      if (rendererSet) {
        glSurfaceView.onResume();
      }
    }

    @Override protected void onPause() {
      super.onPause();
      if (rendererSet) {
        glSurfaceView.onPause();
      }
    }

### Renderer 类

Renderer 类是一个渲染器，其中主要实现了 onSurfaceCreated , onSurfaceChanged , onDrawFrame 三个方法来实现对页面的事件回调。请注意，只能在主线程中调用 OpenGL 的方法。

新建一个 Renderer , 取名为 FirstOpenGLProjectRenderer , 代码如下：

    public class FirstOpenGLProjectRenderer implements Renderer {
        @Override
        public void onSurfaceCreated(GL10 glUnused, EGLConfig config) {
            // Set the background clear color to red. The first component is
            // red, the second is green, the third is blue, and the last
            // component is alpha, which we don't use in this lesson.
            glClearColor(1.0f, 0.0f, 0.0f, 0.0f);
        }

        /**
         * onSurfaceChanged is called whenever the surface has changed. This is
         * called at least once when the surface is initialized. Keep in mind that
         * Android normally restarts an Activity on rotation, and in that case, the
         * renderer will be destroyed and a new one created.
         *
         * @param width
         *            The new width, in pixels.
         * @param height
         *            The new height, in pixels.
         */
        @Override
        public void onSurfaceChanged(GL10 glUnused, int width, int height) {
            // Set the OpenGL viewport to fill the entire surface.
            glViewport(0, 0, width, height);
        }

        /**
         * OnDrawFrame is called whenever a new frame needs to be drawn. Normally,
         * this is done at the refresh rate of the screen.
         */
        @Override
        public void onDrawFrame(GL10 glUnused) {
            // Clear the rendering surface.
            glClear(GL_COLOR_BUFFER_BIT);
        }
    }

在 onSurfaceCreated 方法中 *glClearColor(1.0f, 0.0f, 0.0f, 0.0f);* 设置清空屏幕用的颜色，分别对应红色、绿色和蓝色，最后一个为透明度。

在 onSurfaceChanged 方法中 *glViewport(0, 0, width, height);* 设置了视口尺寸，告诉 OpenGL 可以用来渲染的 surface 的大小。

在 onDrawFrame 方法中 *glClear(GL_COLOR_BUFFER_BIT);* 会擦除屏幕上的所有颜色，并用 *glClearColor* 中的颜色填充整个屏幕。

在使用 OpenGL 的方法时候，可能要在前面加入 *GLES20.* , 为了方便我们可以使用组织导入：
*import static android.opengl.GLES20*

怎么样，是不是超简单？在完成这些后，就可以将程序运行起来了，你将看到一个完全红色的屏幕。下一章才真正开始，我们将学习顶点和着色器。

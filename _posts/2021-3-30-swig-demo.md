---
layout:     post
title:      "Swig Test Demo"
subtitle:   "Keep Going Roger"
date: 2021-03-30 09:00:01 +0800
author:     "Roger"
header-img: "img/home-bg-o.jpg"
tags:
    - swig
---
Swig 小试牛刀
---

### Swig

Swig 是一个 C++ 跨平台的方案库，通过 C++ 文件自动生成跨平台的中间文件，和 Djinni 的区别是 swig 先写 C++ 然后再生成跨平台的文件，而 Djinni 必须是先写好接口文件，然后生成 C++ 的头文件，再继承这个头文件来写 C++ 文件。

Swig 的优势在于支持更多的平台，而且持续更新，上个版本是 20 年六月份，而 Djinni 已经没有维护了。

由于 swig 官网的 demo 时代已经太久远了，所以更新一个自己尝试的版本。

### Setup

[http://www.swig.org/download.html](http://www.swig.org/download.html)

直接下载安装即可

文档地址 [http://www.swig.org/tutorial.html](http://www.swig.org/tutorial.html) 

阅读文档可以发现 swig .i 的这个文件就是用来生成跨平台中间文件

Android Studio 注意配置 NDK 以及 CMAKE 等

### Demo

通过 Android studio 新建一个 C++ 的 Android 工程，注意可能要拉下来一点才能看到。

Android Studio 会自动生成一个 Cpp 文件夹并包含 CMakeLists.txt 和 native-lib.cpp 文件。

先不用管。

在 Cpp 文件夹下我们新家一个 *swigtest.h* 文件，内容如下

	
	#ifndef SWIGTEST_SWIGTEST_H
	#define SWIGTEST_SWIGTEST_H
	#endif //SWIGTEST_SWIGTEST_H
	class swigtest
	{
	public:
	    swigtest() {}
	    ~swigtest() {}
	public:
	    virtual int GetNumber();
	};

再添加一个 *swigtest.cpp* 文件，内容如下

	#include "swigtest.h"
	int swigtest::GetNumber()
	{
	    return 666;
	}

写了一个 swigtest 类，里面有一个 GetNuber 方法，返回 666.

接下来添加一个 *swigtest.i* 文件，用来生成 JNI 相关文件，内容如下

	/* File : example.i */
	%module Test
	%{
	#include "swigtest.h"
	%}
	%include "swigtest.h"

接下来 在 Android studio 的 Terminal 中运行命令

	swig -c++ -java -package com.mycompany.swigtest -outdir src/main/java/com/mycompany/swigtest -o src/main/cpp/swigtest_wrap.cpp src/main/cpp/swigtest.i

可以看到 swig 自动生成了 *cpp/swigtest_wrap.cpp* 文件，在 *src/main/java/com/mycompany/swigtest* 下生成了 

*swigtest.java* （用来调用 JNI 的类，通过实例化这个类来调用 JNI 方法 
*Test.java*（空实现， swig 生成的 java 文件由上层实现的部分 
*TestJNI.java* （实际的 JNI 类，调用到 C++ 方法

接下来修改 CMakeLists.txt 文件

	cmake_minimum_required(VERSION 3.10.2)
	project("swigtest")
	add_library( # Sets the name of the library.
	             native-lib
	             # Sets the library as a shared library.
	             SHARED
	             # Provides a relative path to your source file(s).
	             swigtest.cpp
	             swigtest_wrap.cpp
	             native-lib.cpp
	             )
	find_library( # Sets the name of the path variable.
	              log-lib
	              # Specifies the name of the NDK library that
	              # you want CMake to locate.
	              log )
	target_link_libraries( # Specifies the target library.
	                       native-lib
	                       # Links the target library to the log library
	                       # included in the NDK.
	                       ${log-lib} )

注意 add_library 中需要添加 *swigtest.cpp* *swigtest_wrap.cpp* 并且需要按顺序，不然会报错找不到这个 swigtest 的 C++ 类。

整个工程的下载路径 [https://github.com/Rogero0o/swigtest](https://github.com/Rogero0o/swigtest)

### 总结

Swig Demo 踩坑记录，希望能帮到有缘人。

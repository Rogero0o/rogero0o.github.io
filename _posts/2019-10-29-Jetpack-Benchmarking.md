---
layout:     post
title:      "Jetpack Benchmarking"
subtitle:   "Google I/O 2019 Jetpack Benchmarking"
date: 2019-10-29 11:00:01 +0800
author:     "Roger"
header-img: "img/home-bg-o.jpg"
tags:
    - 翻译
---
Google I/O 2019 Jetpack Benchmarking
=============

> * 原文链接 : [Improving Android app performance with Benchmarking](https://blog.mindorks.com/improving-android-app-performance-with-benchmarking)
> * 原文作者 : [Mindorks](https://blog.mindorks.com/)
> * 译者 : [rogero0o](https://github.com/Rogero0o)


### 代码的测量

除了设备硬件的限制，应用的运行效率也是被使用的算法和代码所影响的，所以选择正确的算法和数据结构是十分重要的。每个程序在算法和数据结构上应该都还有提升的空间。所以，为了让你的应用在数量众多的各种设备上良好的运行，必须确认你的代码优化在各种设备上是生效的。

你不应该写一些程序不需要的代码，同事请不要使用一些不需要的内存。一下介绍一些高效代码的特性：

* 无效的对象：请不要创建无效的对象，因为这样会占用内存，是程序运行变慢。
* 使用静态类：使用静态方法能够获得 15%-20% 速度的提升，所以，如果可以请优先使用静态方法。
* 使用 for-each 循环：对比于使用普通的 loop 或者 别的 loops，推荐使用优化过的 for-each loop 用于 collection 的循环。
* 避免使用 float 对象：尝试不要使用 float 因为他们在 Android 手机上的速度比 integer 慢两倍。

以上就是一些高效率代码的建议，但是如何才能客观的对比代码优化前后的效率呢？有没有什么现成的方法呢？有没有一种可以测量一大段代码耗时时间的工具呢？答案是肯定的，那就是 Benchmarking. 在介绍 Benchmanking 之前，让我们先看看之前是如何测量代码的效率的。

            @Test
            fun codeMeasurement() {
                val worker = TestListenableWorkerBuilder<MyWorker>(context).build() //jetpack workmanager library
                val start = java.lang.System.nanoTime() //for starting time of work
                worker.doWork() //do some work i.e. code to be measured
                val elapsed = (java.lang.System.nanoTime() - start) //time taken to complete the work
                Log.d("Code Measurement", "Time taken was $elapsed ns")
            }


在这里，我们计算代码运行时间的方法是结束时间减去开始时间，但是问题在于，因为硬件和软件的问题，每次运行得出的结果都是不一样的，为了避免这个影响，我们会测量多次，从而获得代码运行的平均时间。

            @Test
            fun codeMeasurement() {
                val worker = TestListenableWorkerBuilder<MyWorker>(context).build() //jetpack workmanager library
                val COUNT = 5 //some iteration count
                val start = java.lang.System.nanoTime() //for starting time of work
                for (i in 0..COUNT) {
                    worker.doWork() //do some work i.e. code to be measured
                }
                // include outliers
                val elapsed = (java.lang.System.nanoTime() - start) / COUNT //average time taken to complete the work
                Log.d("Code Measurement", "Time taken was $elapsed ns")
            }


以上代码，我们使 COUNT 等于 5 次来计算 5 次运行后的平均时间，但是为什么是 5 次？为什么不能是别的次数呢？同样，这样的计算也会有超出平均值的快慢，或是被任何运行在后台的程序所影响。

以上两个示例同时表明，测量代码的运行效率是十分困难的，因为计算平均时间需要找出合适的运行的次数，那么这个 COUNT 的值到底是多少才是合适的呢？这是十分棘手的。那么这些运行代码效率的步骤，就叫做 Benchmarking，现在让我们来看看它的真面目。

### 什么是 Benchmarking?

从上边的段落，我想你已经有了大概的了解。

Benchmarking 可以描述为是用来测量手机对于测量的代码的最快运行时间。通过将手机的状态调整到最后来测量你的应用代码最快的运行效率。

所以我们已经知道我们该如何获取代码运行的平均时间了，但是还是有一些问题，他们是：

* 它是十分不准确的，因为我们在错误的时间测量了错误的事情。

* 这是十分不稳定的，如果我们多次的运行代码那么每次获得的结果都会不同。

* 如果我们使用所有运行次数的平均值，那么在活得结果之前，如何决定需要运行多少次代码来活得这个平均值呢？这是不能决定的。

所以，决定这个运行次数是十分困难的，这就造成了 Benchmark 也是很棘手的。那么有没有办法来找到一段代码实际的运行时间呢？

如果有一个问题，那么肯定就会有解决的办法，那么在这里，我们的解决办法就是 Jetpack Benchmark Library :)

### Jetpack Benchmark Library

在 Google I/O’19， Android 介绍了 Jetpack Benchmark Library 来避免在 Benchmark 的过程中我们遇到的所有问题和困难。

Jetpack Benchmark Libray 是一个用来测量代码效率、避免一些在使用 Benchmark 中常见错误的一个工具。这个 Library 处理了 warmup，测量代码的效率以及在 Android Studio Console 输出结果的功能。

现在，我们之前用于测量的代码可以简化为：

            @get:Rule
            val benchmarkRule = BenchmarkRule()

            @Test
            fun codeMeasurement() {
                val worker = TestListenableWorkerBuilder<MyWorker>(context).build() //jetpack workmanager library
                benchmarkRule.measureRepeated {
                    worker.doWork()
                }
            }

你需要做的就是生成 BenchmarkRule 然后调用 measureRepeated 即可。

让我们来详细看看这个示例，在这里，我们需要测量的是数据库的效率。首先，我们会初始化数据库，接着清空数据库中的表格，然后插入我们需要的测试数据。接下来就是创建我们的测试代码来循环测试数据库查询的效率。

            @get:Rule
            val benchmarkRule = BenchmarkRule()

            @Test
            fun databaseBenchmark() {
                val db = Room.databaseBuilder(...).build()
                db.clearAndInsertTestData()
                benchmarkRule.measureRepeated {
                    db.complexQuery()
                }
            }

但是这里有一些问题，我们的查询是会有缓存的，所以这样我们并不能活得想要的测试结果，所以我们需要在循环中调用 `db.clearAndInsertTestData()` 来清空数据库中的查询缓存，才能得到我们需要的正确的测量值。但是这样的话，我们不仅测量了查询的时间，也测量了清空缓存使用的事件，为了去除这部分多出的时间，我们可以像之前一样计算开始时间和结束时间来获取这部分多余的事件，从而获取最终的测试结果。

            @get:Rule
            val benchmarkRule = BenchmarkRule()

            @Test
            fun databaseBenchmark() {
                val db = Room.databaseBuilder(...).build()
                val pauseOffset = 0L
                benchmarkRule.measureRepeated {
                    val start = java.lang.System.nanoTime()
                    db.clearAndInsertTestData()
                    pauseOffset += java.lang.System.nanoTime() - start
                    db.complexQuery()
                }
                Log.d("Benchmark", "databaseBenchmark_offset: $pauseOffset")
            }

很明显，这样是不优雅的，所以，有一个 API 会用来处理这种情形：`runWithTimeDisabled`.

            @get:Rule
            val benchmarkRule = BenchmarkRule()

            @Test
            fun databaseBenchmark() {
                val db = Room.databaseBuilder(...).build()
                benchmarkRule.measureRepeated {
                    runWithTimeDisabled {
                        db.clearAndInsertTestData()
                    }
                    db.complexQuery()
                }
            }

是不是优雅而干脆？不需要担心更多，使用 Jetpack Benchmark Library 你就能很好的活得 
Benchmarking 结果了，那么如何在 Android Studio 使用呢？让我们看一看。

### 在 Android Studio 中使用

由于 Benchmark Library 还在 alpha 版本，我们需要下载 Android Studio 3.5 及以上版本。我们热爱模块化我们的代码，我们有应用的模块以及库的模块，所以，为了使用 Benchmark library ，需要打开 Android Studio 的配置，以下步骤用于打开 benchmark 模块：

* 下载 Android Studio 3.5 及以上版本。
* Help > Edit Custom Properties.
* 新加一行：npw.benchmark.template.module=true
* 保存退出
* 重启 Android Studio

现在，我们可以创建一个 Benchmark module 了， Benchmark module 模板会自动的配置 
Benchmarking， 以下步骤创建一个 Benchmark module：

* 右键工程 New > Module.
* 选择 Benchmark Module 点击下一步
* 输入 module 名称和语言

使用上面的步骤，一个用于 benchmarking 的 module 就会被创建，包含 benchmark 目录和 
debuggable 设置成 false，这里设置成 false 来避免测试了非正式的代码。

如何运行呢？在 module 里面，找到 benchmark/src/androidTest 然后使用 ctrl+shift+F10 (or cmd+shift+R on mac)。


### Benchmark Configuration

我们使用了一个 Benchmark 的 module，那么它的配置是怎样的呢？让我们来看看：

* Benchmark plugin：帮助拉取 benchmark 报告的库
* Custom runner: 用来稳定 benchmark 的测试。
* Pre-built proguard rules: 优化测试代码和库里的代码

Build.gradle 文件:

        apply plugin: 'com.android.library'
        apply plugin: 'androidx.benchmark'

        android {
            defaultConfig {
                testInstrumentationRunner "androidx.benchmark.junit4.AndroidBenchmarkRunner"
            }

            buildTypes {
                debug {
                    debuggable false
                    minifyEnabled true
                    proguardFiles getDefaultProguardFile(
                            'proguard-android-optimize.txt'),
                            'benchmark-proguard-rules.pro'
                }
            }
        }

        dependencies {
            ...
            androidTestImplementation "androidx.benchmark:benchmark-junit4:1.0.0-alpha05"

        }

并且，在 `AndoridManifest.xml` 文件中，需要做一下设置：

        <!-- Important: disable debuggable for accurate performance results -->
        <application
            android:debuggable="false"
            tools:replace="android:debuggable"/>

这样，我们就配置好了 Benchmarking，测试起来吧！

获取更多相关信息，请阅读原文或参考 Google I/O 2019 的视频 ：）

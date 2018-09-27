---
layout:     post
title:      "Android Accessibility 的少许开发经验"
subtitle:   "Android Accessibility develop tips"
date: 2018-09-21 15:00:01 +0800
author:     "Roger"
header-img: "img/home-bg-o.jpg"
tags:
    - Android Framework
---
Android Accessibility 的少许开发经验
---

### What's Accessibility

简单来说 Accessibility 就是为了让一些残障人士也能正常使用手机或 App 的基本功能，主要包括 Talkback ，视弱的支持等，具体参见：[https://developer.android.com/guide/topics/ui/accessibility/](https://developer.android.com/guide/topics/ui/accessibility/)

当我们的应用发展壮大走向国际的时候，法律条文规定必须支持 Accessibility, 早期的话还能利用 Accessibility 弄一些黑科技（自动抢红包之类的）, 本文只是介绍一些支持 Accessibility 的少许开发经验，并未深入 ：）

### System and Devices

关于 Accessibility 我接触过的会有两个系统版本区别，一个是 三星系统的 Voice Assistant , 再一个是 Google 原生的 Talkback 服务，大体一致，但是在开发时发现两个系统有一些行为上差别还是很大的，具体如下：



### Jcodec

[https://github.com/jcodec/jcodec](https://github.com/jcodec/jcodec)

Jcodec 是一个纯 Java 实现的编码器，功能很强大，可以将一组图片生成 Mp4 文件，但由于是纯 Java 的实现，所以在编解码的效率上十分的低下，而我们关注的不是效率，而是纯 Java 的实现我们是可以将其中间数据取出保存的，这样就完美的实现了我们的需求。

接下来我们要做的就是将 Jcodec 的编码器换成使用 MediaCodec 来编码，再通过 Jcodec 的 Muxer 将 h264 的流写入 mp4 文件中，每写一帧即可将中间数据保存下来，在重启后将存储的中间数据恢复，即可进行下一步操作。

源码可见 [https://github.com/Rogero0o/Mp4MuxerDemo/tree/master](https://github.com/Rogero0o/Mp4MuxerDemo/tree/master)。

有任何疑问或是建议，请直接通过邮箱联系我，欢迎来扰 ：）
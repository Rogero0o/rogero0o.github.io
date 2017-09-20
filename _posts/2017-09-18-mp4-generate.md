---
layout:     post
title:      "Android 追加流生成 Mp4 文件技术方案(支持中断继续)"
subtitle:   "Android Mp4 file generate note"
date: 2017-09-18 12:00:01 +0800
author:     "Roger"
header-img: "img/home-bg-o.jpg"
tags:
    - Android Framework
---
Android 追加流生成 Mp4 文件技术方案（支持中断继续生成）
---

### 背景

Android 中 Mp4 文件的生成主要是通过 Mediacodec 将摄像头采集到的视频从 YUV 格式转成 h264 ，再通过 MediaMuxer 将 h264 的视频流生成 mp4 文件，这个过程就不在赘述了， Google 一搜一大把，其中需要注意的是在将 YUV 数据送入 Mediacodec 中之前需要将 YUV 格式从 NV21 转换成编码器能接收的 I420 格式，建议这个过程使用 JNI 来提高效率， java 的实现方式可以参考一下方法：

	public static void NV21toI420SemiPlanar(byte[] nv21bytes, byte[] i420bytes,
                                            int width, int height) {
        System.arraycopy(nv21bytes, 0, i420bytes, 0, width * height);
        for (int i = width * height; i < nv21bytes.length; i += 2) {
            i420bytes[i] = nv21bytes[i + 1];
            i420bytes[i + 1] = nv21bytes[i];
        }
    }

关于从摄像头采集到转换生成 h264 的过程可以参考文章末尾的示例 Demo。

通过 MediaMuxer 可以将 h264 文件流实时生成 mp4 文件，但是如果中途中断程序，是无法继续生成的，只能从头再来，现在我们需要的就是一种能够在中断后，能够保存临时文件，再次打开程序能够继续添加流并成功生成 Mp4 的技术方案。


### 方案调研

从原理上来说， MediaMuxer 是在调用 *addTrack(MediaFormat format)* 方法的时候将 'csd-0' 和 'csd-1' ，对应 h264 的 sps 和 pps ，简单来说就是视频的一些基本信息，保存下来，接下来就可以通过 *MediaMuxer.writeSampleData()* 方法向 MP4 文件里边写数据，在写入结束后调用 stop() 和 release() 关闭和释放资源。

这里要注意，在调用 stop 的时候，是将 sps , pps ,每个关键帧的偏移量，以及每个关键帧的大小等这些关键的数据参数写入 mp4 文件中，生成寻址表等一些类似头文件的基本信息，这样才完成了 mp4 文件的生成。

所以看到这里我们应该有一个基本思路，只要将 sps,pps,每个关键帧偏移量和大小等等，这些用于生成的关键数据缓存下来，在应用重启之后即可恢复数据继续生成，或是继续往里面写数据，之后再选择生成，这些功能都能完美支持。但是在原生的 MediaMuxer 中，这些中间参数是无法读取保存的，所以我们只能另寻他路，通过 Google 不断的搜寻，找到了一个完美支持的库。

### Jcodec

[https://github.com/jcodec/jcodec](https://github.com/jcodec/jcodec)

Jcodec 是一个纯 Java 实现的编码器，功能很强大，可以将一组图片生成 Mp4 文件，但由于是纯 Java 的实现，所以在编解码的效率上十分的低下，而我们关注的不是效率，而是纯 Java 的实现我们是可以将其中间数据取出保存的，这样就完美的实现了我们的需求。

接下来我们要做的就是将 Jcodec 的编码器换成使用 MediaCodec 来编码，再通过 Jcodec 的 Muxer 将 h264 的流写入 mp4 文件中，每写一帧即可将中间数据保存下来，在重启后将存储的中间数据恢复，即可进行下一步操作。

源码可见 [https://github.com/Rogero0o/Mp4MuxerDemo/tree/master](https://github.com/Rogero0o/Mp4MuxerDemo/tree/master)。

有任何疑问或是建议，请直接通过邮箱联系我，欢迎来扰 ：）
---
layout:     post
title:      "本地编译android4.4.2 Settings环境搭建记录"
subtitle:   "源码解析"
date: 2015-04-16T16:22:56+00:00
author:     "Roger"
header-img: "img/android-bg.jpg"
tags:
    - Android Framework
---
最近在本地编译Settings环境的搭建上走了点弯路，现在记录一下，希望能帮到有需要的同学。
  
1.解除ADT对android内部API的使用限制：
  
进入 eclipse的plugins文件夹，找出名为com.android.ide.eclipse.adt_\*.jar的文件。做一个备份（以防修改错了），另外复制一份改文件到一个单独的&#8221;experimental”文件夹，在那里进行字节码修改。修改\*.jar为*.zip，解压文件到一个单独的文件夹，下面就是我所得到的：
  
[<img class="alignnone size-medium wp-image-73" src="http://2.rogerbolg.sinaapp.com/wp-content/uploads/2015/04/adt_plugins-300x109.png" alt="adt_plugins" width="300" height="109" />](http://2.rogerbolg.sinaapp.com/wp-content/uploads/2015/04/adt_plugins.png)
  
进入到com/android/ide/eclipse/adt/internal/project子目录，找出AndroidClasspathContainerInitializer.class文件。
  
用notepad++ 打开，将&#8221;com/android/internal/\*\*”替换为&#8221;com/android/internax/\*\*”。
  
修改完后，保存文件，zip压缩文件夹，文件名和原始版本一样。然后重命名为.jar。替换原来的文件。重启ADT。

2.设置工程
  
根据Android.mk，找出需要的库文件，比如：
  
LOCAL\_STATIC\_JAVA_LIBRARIES := com.android.phone.common
  
在源码下找到out/target/common/obj/JAVA\_LIBRARIES/com.android.phone.common\_intermediates/classes.jar
  
，加入工程，其他需要的库也加入,直到编译正确。
  
一个一个加入，最后如图：
  
[<img class="alignnone size-full wp-image-74" src="http://2.rogerbolg.sinaapp.com/wp-content/uploads/2015/04/jars1.jpg" alt="jars1" width="291" height="287" />](http://2.rogerbolg.sinaapp.com/wp-content/uploads/2015/04/jars1.jpg)

调整包的顺序至前面，最后如图：
  
[<img class="alignnone size-medium wp-image-75" src="http://2.rogerbolg.sinaapp.com/wp-content/uploads/2015/04/jars2-300x240.jpg" alt="jars2" width="300" height="240" />](http://2.rogerbolg.sinaapp.com/wp-content/uploads/2015/04/jars2.jpg)

其中有一个包含内部API的jar包，就是图中android4.4.2中的jar包，下载地址：
  
<a title="http://download.csdn.net/detail/baimingyong007/7979321" href="http://download.csdn.net/detail/baimingyong007/7979321" target="_blank">http://download.csdn.net/detail/baimingyong007/7979321</a>

全部jar包下载地址：
  
<a title="http://download.csdn.net/detail/baimingyong007/7979321" href="http://download.csdn.net/detail/baimingyong007/7979321" target="_blank">http://download.csdn.net/detail/fly_o0o/8601019</a>

导入后还有一两个内部类无法识别，注释即可。
  
完成后即可编译出APK
  
<!--more-->

3、使用目标系统的platform密钥来重新给apk文件签名。这步比较麻烦，
  
首先找到密钥文件，在我的Android源码目录中的位置
  
是&#8221;build\target\product\security&#8221;，如果android:sharedUserId=&#8221;android.uid.system&#8221;，公匙密匙分别是platform.pk8和platform.x509.pem两个文件，其他类似。
  
然后用Android提供的Signapk工具来签名，signapk.jar在out/host/linux-x86/framework下
  
用法为&#8221;java -jar signapk.jar platform.x509.pem platform.pk8 input.apk output.apk&#8221;，
  
例如：java -jar signapk.jar platform.x509.pem platform.pk8 Settings.apk Settings-signed.apk
  
文件名最好使用绝对路径防止找不到，input.apk和output.apk不要相同，会报错。

4、安装APK
  
运行
  
Adb root
  
Abd adb remount -o rw
  
Adb adb shell &#8220;cd /system/app;rm Settings.apk;&#8221;
  
将原装的Settings卸载，然后点击签名后的APK安装即可。

打开新安装的Settings，在ADT的DDMS进程列表中找到settings，打上绿色小虫子，是不是可以debug了？少年，开始战斗吧。
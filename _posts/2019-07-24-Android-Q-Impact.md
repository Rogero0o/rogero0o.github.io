---
layout:     post
title:      "Android Q Impact"
subtitle:   "What's new in Android Q"
date: 2019-07-24 11:00:01 +0800
author:     "Roger"
header-img: "img/home-bg-o.jpg"
tags:
    - Android Framework
---
Android Q Impact
---

一年一度的 Android 系统又更新啦，今年来到了 Android Q，个人感觉现在 Android 的发展是对应用的限制越来越严格，新的亮眼的功能基本是没有了，略显失望！

### Agenda

1. Scoped Storage
2. Restrictions to background activity starts
3. Updates to non-SDK interface restrictions
4. Guestral Navigation
5. Dark Theme
6. Sharing improvements
7. Bubbles
8. Smart Reply


前三点是新系统新增的对权限和隐私的限制，后面六点为新增的功能，下面将分别讲解。

### 权限及隐私

####  Scoped Storage

[https://developer.android.com/preview/privacy/scoped-storage](https://developer.android.com/preview/privacy/scoped-storage)

简单来说，如果将 targetsdkversion 改为 29 的话，系统将不允许应用访问外部存储的问题，例如相机拍摄的文件以及 Download 文件夹，需要使用 MediaStore 的 openFile 来获取文件流，否则直接报 Filenotfound error。 'MediaStore.MediaColumns.DATA' 这个字段也被标识过期，因为无法直接使用 file path 这种方式打开文件。

这个改动的影响是非常大的，凡是涉及到文件拷贝，图库以及相机等功能的代码都可能被影响，所以 Android Q 提供了一个过渡方案，在 manifest 中设置 ‘android:requestLegacyExternalStorage="true"’ 来屏蔽 Scoped Storage 生效， but 明年应该就不行了，所以早做早好吧.

#### Restrictions to background activity starts

[https://developer.android.com/preview/privacy/background-activity-starts](https://developer.android.com/preview/privacy/background-activity-starts)

出于不希望应用直接打断用户当前操作的目的，Android Q 开始严格限制应用从后台启动前台 Activity，对于需要显示 Incoming Call 或是闹铃提醒的应用，Android 推荐使用 FullScreen Intent 来启动一个 HeadsUp Notification，注意在 Android Q 上需要新申请一个 USE_FULL_SCREEN_INTENT 的 permission.

#### Updates to non-SDK interface restrictions

[https://developer.android.com/preview/non-sdk-q](https://developer.android.com/preview/non-sdk-q)

这是谷歌从 Android P 就开始限制的功能，不能调用非 Sdk 提供的接口，使用类似反射调用的方法将直接报错，这对维护系统稳定是很有帮助的。


### 新功能

#### Guestral Navigation

[https://developer.android.com/preview/features/gesturalnav](https://developer.android.com/preview/features/gesturalnav)

现在 Android Q 系统提供新的手势滑动功能，新增 edge-to-edge 的手势滑动来代替 back 键，但是这个在左滑退出时会和 Drawer 划出菜单栏冲突，变成无法滑动划出 Drawer，系统提供 'setSystemGestureExclusionRects(exclusionRects)' 方法来屏蔽特定区域的手势滑动，需要自己自定义适配，系统在 Beta 5 的时候也提供了在左边缘长按划出 Drawer 的功能，可以根据需要进行修改。

#### Dark Theme

[https://developer.android.com/preview/features/darktheme](https://developer.android.com/preview/features/darktheme)

Android Q 系统提供了 Dark Theme，如果你的颜色值等是完全是使用系统的，则直接使用系统提供的 Dark theme 则能直接适配黑暗模式，but 一般应用都应该有自定义的颜色和图片，所以适配 Dark Theme 还是很有工作量的。


#### Sharing improvements

[https://developer.android.com/preview/features/sharing](https://developer.android.com/preview/features/sharing)

Android 系统的分享页面已经更新了，支持直接分享给指定联系人和 Sharing 内容的预览。应用层不需要做任何改动。

#### Bubbles

[https://developer.android.com/preview/features/bubbles](https://developer.android.com/preview/features/bubbles)

Bubbles 是 notification 的一个延伸，支持用户直接打开图标上的小红点并进行交互。然而， “Note: Bubbles are currently enabled for all users in the Q developer previews. In the final release, Bubbles will be available for developer use only.” 所以感觉还是在开发期的一个功能。

#### Smart Reply

[https://developer.android.com/preview/features](https://developer.android.com/preview/features) 页面末尾

Smart Reply 是 Android P 上就支持的一个功能，但是在 P 系统上无法自动生成 reply 的建议，在 Android Q 系统上通过一个系统内置的可离线使用的大数据分析系统（我猜的）来实现自动生成 suggest 的 reply 建议，如果你的 notification 已经适配过 P （既能在 Notificaiton 上一直的 reply），那么不需要做任何改动，在 Android Q 上就能自动生成 Smart Reply 的 button。
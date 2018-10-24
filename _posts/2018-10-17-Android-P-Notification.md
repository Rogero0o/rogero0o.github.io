---
layout:     post
title:      "Android P Notification 那些你不知道的坑"
subtitle:   "The new strange behaviors in Android P Notification "
date: 2018-10-17 12:00:01 +0800
author:     "Roger"
header-img: "img/home-bg-o.jpg"
tags:
    - Android Framework
---
Android P Notification 那些你不知道的坑
---

Android Pie 已经发布许久，相信大家已经做过了 Android P 版本的适配，如果不升级 Target SDK 来说的话，问题不大，基本没有什么工作量。但若是需要升级 Target SDK 到 28 的话，如果是 MESSAGE 的类型的 Notification 是有一些官方文档没提到的坑，在这总结一下：

1. 在 Android P 中，*NotificationCompat.Builder.setLargeicon()* 方法设置的不再是对话人物的头像，而是这个 Notification 在缩小状态下才会显示出来的属于这个 Notification 的头像。对话人物的头像现在统一由 Person 类来设置。

2. 对话人物头像的设定现在不仅仅可以设置对方，还可以设置自己回复后的头像，在生成 messagingStyle 的时候将代表自己的 Person 类作为构造参数传入。而后在回复后需要更新 notification 时调用 messagingStyle.addMessage()，这时的 Person 传入 Null 即代表这条信息是自己回复的。

3. 在 Android P 中，如果一个 Notification 具有快速回复的功能，那么在快速回复之后，调用 cancel 方法取消这个 notification 是无效的，官方的说法是这是一个 Android P 的功能，参考过 P 的 Message 应用的确也是不消失的。

4. 由于快速回复后 notification 是不消失的，所以在回复成功后，需要调用 notification 对应的 builder 再去更新 messagingStyle 来刷新 notification， 这时请注意将 builder 的对象保持持有，因为如果 builder 被回收的话，问题不仅仅是回复信息后无法更新 notification，而且有可能在收到同样的 notification 时完全就显示不出来，因为 builder 已经被销毁了。

5. 为了解决第四点中提到的问题，我们都会持有 builder 的对象，但是如果在收到 notification 的时候用户将应用手动 kill 掉，那么我们持有的 builder 又被销毁，这时在更新时正确的做法是获取 notification 的实例，然后通过其中的参数再 recreate 一个相同的 builder。

6. 请注意，再次生成或从 cache 获取的 builder 是需要再次设置 contentIntent 的，不然会造成无法跳转。

7. 如果需要从 notificaiton 回复的时候不需要声音，那么在设置 notificaitonChannel 的时候需要调用 channel.setSound(null,null) , 从 notificaiton 或是 builder 里边设置是没有效果的。


源码可见 [https://github.com/Rogero0o/AndroidP_Notification_Demo](https://github.com/Rogero0o/AndroidP_Notification_Demo)。

有任何疑问或是建议，请直接通过邮箱联系我，欢迎来扰 ：）

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

### System and Devices Different

关于 Accessibility 我接触过的会有两个系统版本区别，一个是 三星系统的 Voice Assistant , 再一个是 Google 原生的 Talkback 服务，大体一致，但是在开发时发现两个系统有一些行为上差别还是很大的，三星也可以选择使用 Google 的 Talkback，两者的差别具体如下：

    a. 三星中 Viewpage 页面中右滑，会定位到不可见的上下页面的元素，可通过监听滑动，然后设置对 Accessibility 是否可见解决。

    b. 三星中定位到 Webview 会读出网页中的内容，而 Talkback 只会读网页的 Title 加上 “webview”，这个需要根据需求适配。

    c. 三星中打开新页面不会自动定位到新页面的第一个元素，相比之下 Talkback 是会的，并且双击屏幕还能出发前页面的点击效果，这点三星也是挺坑的。

    d. Listview 滑动停止时三星会有一个 showing item %s of %s, Talkback 不会，这个需要根据需求自己修改。

    e. 从外部进入 listview 的时候 Talkback 会在末尾加一个 “in list” ，三星的不会。


### 一些问题以及解决方法


1. ListView 在计算 count 的时候会将 header 和 footer 都计算在内，如果有上下拉刷新这种不可见的 header 和 footer 的话问题就更大了，读出的数量往往和可见的正确数量相差特别大，这时候需要对 listview 的 AccessibilityNodeInfo 做一些初始化的修改。在谷歌原生的手机上，当进入 listview 的时候系统会默认读一个 “in list， %s items” 这里边的 %s 也是当前 listview 的总数，这个总数也很有可能是错误的，处理这些错误的方法如下：


	   mListView.setAccessibilityDelegate(new View.AccessibilityDelegate() {

            @TargetApi(Build.VERSION_CODES.LOLLIPOP)
            @Override
            public void onInitializeAccessibilityNodeInfo(View host, AccessibilityNodeInfo info) {
                super.onInitializeAccessibilityNodeInfo(host, info);
                if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.LOLLIPOP) {
                    AccessibilityNodeInfo.CollectionInfo old = info.getCollectionInfo();
                    info.setCollectionInfo(AccessibilityNodeInfo.CollectionInfo.obtain(
                            old.getRowCount() - mListView.getHeaderViewsCount()
                                    - mListView.getFooterViewsCount(),
                            old.getColumnCount(),
                            old.isHierarchical(),
                            old.getSelectionMode()));
                }
            }

            @Override
            public void onInitializeAccessibilityEvent(View host, AccessibilityEvent event) {
                super.onInitializeAccessibilityEvent(host, event);
                event.setFromIndex(-1);
                event.setToIndex(-1);
                event.setItemCount(-1);
            }
        });

我们通过重写 onInitializeAccessibilityNodeInfo 方法来控制 info 的 CollectionInfo 中的数量，减去 headerView 和 footerView 的数量后，这样三星上读出来的数量就正常了。

通过重写 onInitializeAccessibilityEvent 这个方法可以避免在划入 listview 的时候谷歌的系统读出 “in list %s items”，这样也避免了 items 总数的不对。



2. 当使用一些系统的 Dialog 的时候，一打开 Dialog 的时候可能会默认读一遍 Dialog 的 Title 内容，然后自动聚焦到 Ttile 的时候又读了一遍，造成了重复的问题，在 Dialog 中复写一下方法能避免该问题：



      	   @Override
          public boolean dispatchPopulateAccessibilityEvent(@NonNull AccessibilityEvent event) {
              if (event.getEventType() == AccessibilityEvent.TYPE_WINDOW_STATE_CHANGED) {
                  return true;
              }
              return super.dispatchPopulateAccessibilityEvent(event);
          }

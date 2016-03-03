---
id: 191
title: Android Support Library 23.2
date: 2016-02-25T12:00:19+00:00
author: roger
layout: post
views:
  - 1305
duoshuo_thread_id:
  - 6255073630967825154
categories:
  - 所有文章
  - 技术
  - 首页
---
原文地址：<a title="Link" href="http://android-developers.blogspot.jp/2016/02/android-support-library-232.html" target="_blank">Link</a>.
  
第一次翻译,若有错误请指出。:)

**Android Support Library 23.2**

当我们说起 Android Support Library 时，必须清楚的认识到这不仅仅只是一个库，而是一个能对API向后兼容的，提供独特的功能而不需要最新平台(plathform)支持的 库的集合。23.2版本在许多现有库的基础上增加一些新的支持以及功能。

**Support Vector Drawables and Animated Vector Drawables**

定义在XML里的矢量图片能替换大量的png图片资源，之前只有棒棒糖以及更高版本才能支持的功能，现在通过两个新增支持库 support-vector-drawable 和 support-animated-vector-drawable，分别提供了 VectorDrawable(矢量图片) 和 AnimatedVectorDrawable(矢量图片动画) 两项功能。

Android studio 1.4 通过在编译期间生成png图片来实现对矢量图的支持。关闭这项功能（以获得这一支持库的真正优势和节省空间），你需要在你的 build.gradle 文件添加代码：

<div class="codecolorer-container xml twitlight" style="overflow:auto;white-space:nowrap;width:100%;">
  <div class="xml codecolorer">
    vectorDrawables.useSupportLibrary = true
  </div>
</div>

在 gradle2.0 及以上你可以像这样写

<div class="codecolorer-container xml twitlight" style="overflow:auto;white-space:nowrap;width:100%;">
  <div class="xml codecolorer">
    // Gradle Plugin 2.0+<br /> android {<br /> &nbsp; &nbsp; defaultConfig {<br /> &nbsp; &nbsp; &nbsp; &nbsp; vectorDrawables.useSupportLibrary = true<br /> &nbsp; &nbsp; }<br /> }
  </div>
</div>

如果是 gradle1.5，则需要这样写

<div class="codecolorer-container xml twitlight" style="overflow:auto;white-space:nowrap;width:100%;">
  <div class="xml codecolorer">
    // Gradle Plugin 1.5<br /> android {<br /> &nbsp; &nbsp; defaultConfig {<br /> &nbsp; &nbsp; &nbsp; &nbsp; generatedDensities = []<br /> &nbsp; &nbsp; } <br /> <br /> <br /> // This is handled for you by the 2.0+ Gradle Plugin<br /> &nbsp; &nbsp; aaptOptions {<br /> &nbsp; &nbsp; &nbsp; &nbsp; additionalParameters "--no-version-vectors"<br /> &nbsp; &nbsp; }<br /> }
  </div>
</div>

VectorDrawableCompat 最低支持到 API7 ， AnimatedVectorDrawableCompat 最低支持到 API 11 。受限于android的图片加载机制，不是所有支持图片id的地方（例如xml文件里）都能使用矢量图。值得庆幸的是，兼容包（AppCompat）已经增加了许多功能，可以很容易地使用新的矢量绘图资源。

首先，当我们在ImageView（或者子类例如 ImageButton 和 FloatingActionButton）中使用兼容包时，你将会使用到新属性 app:srcCompat 来指定矢量图（所有 android:src 能够指定的图片 app:srcCompat都能指定）

<div class="codecolorer-container text twitlight" style="overflow:auto;white-space:nowrap;width:100%;">
  <div class="text codecolorer">
    &nbsp;<ImageView &nbsp;<br /> &nbsp; android:layout_width="wrap_content" &nbsp;<br /> &nbsp; android:layout_height="wrap_content" &nbsp;<br /> &nbsp; app:srcCompat="@drawable/ic_add" />
  </div>
</div>

如果需要在运行时切换图片，使用和之前一样的方法 setImageResource()。使用兼容包和 app:srcCompat是将矢量图整合进APP中最简单有效的方法。

你会发现在棒棒糖中不使用 app:srcCompat 引用矢量图的方法都会失败。然而，当矢量图被 另一个 drawable容器（例如 StateListDrawable, InsetDrawable, LayerDrawable, LevelListDrawable,和 RotateDrawable）引用时，兼容包是可以加载该矢量图的。所以，在一些无法直接使用兼容包（直接使用app:srcCompat）的地方，可以通过间接的方式来使用矢量图，例如 TextView 的 android:drawleLeft 属性。
  
<!--more-->

**AppCompat DayNight theme**

对矢量图的支持对兼容包已经是一个巨大的改变，在这个版本还增加了一个新的主题（Theme）：Theme.AppCompat.DayNight.

    <img class="alignnone size-medium wp-image-195" src="http://2.rogerbolg.sinaapp.com/wp-content/uploads/2016/02/image05-169x300.png" alt="image05" width="169" height="300" /><img class="alignnone size-medium wp-image-194" src="http://2.rogerbolg.sinaapp.com/wp-content/uploads/2016/02/image01-169x300.png" alt="image01" width="169" height="300" />

在API14之前，DayNight主题以及相关的 DayNight.NoActionBar, DayNight.DarkActionBar, DayNight.Dialog 等等，提供了相同的亮度（意思是不支持API14之前？）。但是 API14 及 以后的设备，DayNight主题支持应用切换 白天 和 夜晚 主题，根据 是否为 ‘夜晚’ 决定是否从白天主题有效的切换到夜晚主题。

默认的，是否为 ‘夜晚’ 是由系统值（由方法 UiModeManager.getNightMode()取得）决定的，但是可以通过重写AppCompatDelegate中的方法来设置是否为‘夜晚’。您可以在app的整个生命周期（除非进程重启）设置这个默认值，设置该值的方法一种是通过静态方法 AppCompatDelegate.setDefaultNightMode() ，或者是 通过getDelegat()取得一个AppCompatDelegate 对象，然后调用方法setLocalNightMode() 来设置当前 activity 或者 dialog 的主题。

当使用 AppCompatDelegate.MODE\_NIGHT\_AUTO 值时，您的手机时间和您最后定位的地点（如果您的手机开启了定位许可）将会被用于切换白天和黑夜的依据， MODE\_NIGHT\_NO 和 MODE\_NIGHT\_YES 则分别强制设定了从不或是一直使用夜晚主题。

当使用白天主题时，彻底的测试是非常重要的，因为白天主题的亮度很可能导致一些文字或者图标变得不可阅读。如果将你的主题中标准的 TextAppearance.AppCompat 样式（例如 android:textColorPrimary）设置到文字或是颜色中，你将会发现他们已经自动更新了。

尽管如此，当你需要为夜晚主题定制一些特殊化资源时，兼容包会重新加载 夜晚专用资源文件夹（night resource qualifier folder），让定制所有资源变得可能。请考虑使用标准色彩，或是遵循兼容包所推荐的色彩方案，这样能让使用这个主题变得更加简单。

**Design Support Library: Bottom Sheets**

设计支持库提供了许多 Material Design 的格局，这个版本提供了对 bottom sheets 的支持。

[<img src="http://2.rogerbolg.sinaapp.com/wp-content/uploads/2016/02/image06.png" alt="image06" width="360" height="640" class="aligncenter size-full wp-image-196" />](http://2.rogerbolg.sinaapp.com/wp-content/uploads/2016/02/image06.png)

在 CoordinatorLayout （即增加属性 app:layout_behavior=”android.support.design.widget.BottomSheetBehavior” ）的子view附上一个 BottomSheetBehavior ，你将会得到五个状态的触摸回调。
  
1.STATE\_COLLAPSED，这个关闭的状态是默认的，沿着父视图的下沿显示一小部分，显示的高度可以通过 app:behavior\_peekHeight 属性设置，默认是0
  
2.STATE_DRAGGING，中间状态，表示用户正在打开或者关闭抽屉（sheet）
  
3.STATE_SETTLING，抽屉从被放开运行到最终位置的状态
  
4.STATE_EXPANDED，抽屉被完全打开的状态，即抽屉的高度完全显示出来（当抽屉高度小于主视图时）或是主视图被抽屉完全充满时。
  
5.STATE\_HIDDEN，抽屉完全不可见的默认状态，（app:behavior\_hideable 属性可以设置），打开这个允许用户向下滑动直至完全关闭抽屉。

如果你的抽屉（sheet）中有需要滑动的试图，请务必确保它能支持嵌套滑动（例如 NestedScrollView, RecyclerView, or ListView/ScrollView on API 21+）。

如果你需要接受状态回调，可以添加一个 BottomSheetCallback:

<div class="codecolorer-container java twitlight" style="overflow:auto;white-space:nowrap;width:100%;">
  <div class="java codecolorer">
    <span class="co1">// The View with the BottomSheetBehavior &nbsp;</span><br /> &nbsp;<a href="http://www.google.com/search?hl=en&q=allinurl%3Aview+java.sun.com&btnI=I%27m%20Feeling%20Lucky"><span class="kw3">View</span></a> bottomSheet <span class="sy0">=</span> coordinatorLayout.<span class="me1">findViewById</span><span class="br0">&#40;</span>R.<span class="me1">id</span>.<span class="me1">bottom_sheet</span><span class="br0">&#41;</span><span class="sy0">;</span> &nbsp;<br /> &nbsp;BottomSheetBehavior behavior <span class="sy0">=</span> BottomSheetBehavior.<span class="me1">from</span><span class="br0">&#40;</span>bottomSheet<span class="br0">&#41;</span><span class="sy0">;</span> &nbsp;<br /> &nbsp;behavior.<span class="me1">setBottomSheetCallback</span><span class="br0">&#40;</span><span class="kw1">new</span> BottomSheetCallback<span class="br0">&#40;</span><span class="br0">&#41;</span> <span class="br0">&#123;</span> &nbsp;<br /> &nbsp; &nbsp; @Override &nbsp;<br /> &nbsp; &nbsp; <span class="kw1">public</span> <span class="kw4">void</span> onStateChanged<span class="br0">&#40;</span>@NonNull <a href="http://www.google.com/search?hl=en&q=allinurl%3Aview+java.sun.com&btnI=I%27m%20Feeling%20Lucky"><span class="kw3">View</span></a> bottomSheet, <span class="kw4">int</span> newState<span class="br0">&#41;</span> <span class="br0">&#123;</span> &nbsp;<br /> &nbsp; &nbsp; &nbsp; <span class="co1">// React to state change &nbsp;</span><br /> &nbsp; &nbsp; <span class="br0">&#125;</span> &nbsp;<br /> &nbsp; &nbsp; &nbsp; @Override &nbsp;<br /> &nbsp; &nbsp; &nbsp; <span class="kw1">public</span> <span class="kw4">void</span> onSlide<span class="br0">&#40;</span>@NonNull <a href="http://www.google.com/search?hl=en&q=allinurl%3Aview+java.sun.com&btnI=I%27m%20Feeling%20Lucky"><span class="kw3">View</span></a> bottomSheet, <span class="kw4">float</span> slideOffset<span class="br0">&#41;</span> <span class="br0">&#123;</span> &nbsp;<br /> &nbsp; &nbsp; &nbsp; &nbsp;<span class="co1">// React to dragging events &nbsp;</span><br /> &nbsp; &nbsp;<span class="br0">&#125;</span> &nbsp;<br /> &nbsp;<span class="br0">&#125;</span><span class="br0">&#41;</span><span class="sy0">;</span>
  </div>
</div>

此版本不仅提供了 BottomSheetBehavior 获取抽屉状态的回调，而且还提供了 BottomSheetDialog 和 BottomSheetDialogFragment 来覆盖所有应用场景。只需要简单的替换成 AppCompatDialog 或 AppCompatDialogFragment ，效果等于使用普通dialog并使用抽屉主题。

**Support v4: MediaBrowserServiceCompat**

Support v4 库为许多支持库提供了基础，并且为一些新版本介绍的特征提供支撑(backports).

之前发布了一个播放媒体的可靠的工具类 MediaSessionCompat， 在这之上，这个版本添加了 MediaBrowserServiceCompat 和 MediaBrowserCompat ，使得支持的版本扩展到 支持API4及其之后版本。提供了这个标准的接口,使得Service及UI界面与媒体的连接更加紧密,无论在Android设备 还是 Android Wear 上播放媒体都变得更加的便捷。

**RecyclerView**

RecyclerView 提供了先进、灵活的创建列表和网格的功能，并且支持动画。这个版本在 LayoutManager API中提供了一个令人兴奋的特性：auto-measurement.这使得 RecyclerView 可以根据内容的大小来决定自身的大小。这使得RecyclerView之前不能使用的属性（例如WRAP_CONTENT）,现在可以使用了。现在你会发现在LayoutManagers 中构建的控件现在都支持 auto-measurement了.

基于这个改变，请确保你的item的布局属性，之前忽略的布局属性（例如 在滑动方向上的MATCH_PARENT），现在将会完全的展开。

如果你自定义了一个LayoutManager并且不是之前的拓展，你可以调用 setAutoMeasureEnabled(true) 以及做一些细小的变化（详情请见Javadoc）来支持这项特性。

请注意，尽管 RecyclerView 可以设置子布局的动画，但是动画不会改变RecyclerView 自己的位置。如果你需要RecyclerView随着动画移动，请查看 Transition APIs.

**Custom Tabs**

Custom Tabs 可以在保持你的app外观，在用户毫无知觉的情况下无缝的过度到 web 内容。随着这项功能的发布，现在可以在底栏增加一个在侧边显示网站内容的动作。

<img src="http://2.rogerbolg.sinaapp.com/wp-content/uploads/2016/02/image04.png" alt="image04" width="360" height="640" class="aligncenter size-full wp-image-198" />

随着新加的方法 addToolbarItem()，你现在可以在底栏增加五种动作，并且一旦会话开始就可以使用 setToolBarItem()更新他们。和之前的setToolbarColor()类似，你同样会发现一个setSecondaryToolbarColor()方法来自定义底栏的背景颜色。

**Leanback for Android TV**

Leanback Library 提供一个你需要的工具来方便的将你的应用适配到 Android TV 中，通过针对电视体验优化了的标准组件。GuidedStepFragment 在此版本得到了显著的改进。

<img src="http://2.rogerbolg.sinaapp.com/wp-content/uploads/2016/02/image02.png" alt="image02" width="640" height="360" class="aligncenter size-full wp-image-199" />

最明显的变化可能是引入了一个两列可用的按钮动作（通过重写onCreatButtonActions()或者调用setButtonActions()添加的）。这使得到达列表底部更加简单，而不需要穿越整个list。

说到GuideActions，有一些新的功能，支持更丰富的输入，包括可编辑的描述（通过 descriptionEditable()方法），下拉动作的子动作（通过subActions()），还有 GuidedDatePickerAction。

<img src="http://2.rogerbolg.sinaapp.com/wp-content/uploads/2016/02/tv_combined_image.png" alt="tv_combined_image" width="640" height="360" class="aligncenter size-full wp-image-200" />

当你需要时,这些组件让你更容易获取用户的意图。
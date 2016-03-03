---
id: 156
title: 《Android 开发艺术探索》 读书笔记
date: 2015-11-30T10:47:07+00:00
author: roger
layout: post
views:
  - 219
duoshuo_thread_id:
  - 6222770357506409217
categories:
  - 所有文章
  - 技术
  - 首页
---
最近在研读任教主的《Android开发艺术探索》大作，真是一本好书，以下为记录的读书笔记：

第一章 Activity的生命周期和启动模式
  
1.用户打开新的activity或者切换到桌面的时候：onPause->onStop;
  
特殊情况：如果新activity采用了透明主题，那么当前activity不会回调onStop；
  
2.onStart和onStop,onResume和onPause的区别是什么？
  
onStart是从activity是否可见这个角度回调的，后者是从activity是否位于前台这个角度考虑的，没有其他明显区别
  
3.standard模式是activity启动的默认模式，在ApplicationContext中启动standard模式的activity会报错，因为standard模式的activity默认会进入启动它的activity所属的任务栈中。
  
4.singleTask模式的activity切换到栈顶会导致在他之上的栈内的activity出栈
  
5.如果要为intent指定完整的data，必须要调用setDataAndType方法。

第二章 IPC机制
  
1.在一个应用中使用多进程的方法是给四大组件的AndroidMenifest中指定android:process，除此之外别无他法。
  
2.android:process 属性中，进程名以：开头的进程属于当前应用的私有进程。其他应用的组件不可以和它跑在同一个进程中。
  
3.android为每一个应用分配了一个独立的虚拟机，或者说为每一个进程都分配一个独立的虚拟机，不同的虚拟机在内存分配上有不同的地址空间，这就导致在不同的虚拟机中访问同一个类的对象会产生多份副本。
  
4.多进程使用会造成如下几个方面影响：
  
a.静态成员和单例模式完全失效
  
b.线程同步机制完全失效
  
c.SharedPreferences的可靠性下降
  
d.Application会多次创建
  
5.同一个应用间的多进程，本质相当于两个不同的应用采用了SharedUID的模式。
  
6.Serializable接口中SerialVersionUID是用来辅助序列化和反序列化过程的，原则上序列化后的数据中的seriaVersionUID只有和当前类的serialVersionUID相同才能够正常地被反序列化。
  
7.在android中选择用Parcelable效率比较高，序列化到存储设备中或者将对象序列化后通过网络传输推荐用Serializable
  
8.android中的ipc方式：1.使用bundle 2.使用文件共享 3.使用messenger 4.使用AIDL
  
9.使用AIDL实现listener时，需要使用RemoteCallbackList来解注册
  
10.ContentProvider中，除了onCreate由系统回调并运行在主线程里，其他五个方法均由外界回调并运行在Binder线程池中。
  
11.ContentProvider中的增删改查四大方法是存在多线程并发访问的，因此要做好线程同步，可以只使用一个SQLiteDatabase来同步，使用多个则要注意线程同步问题。
  
12.使用Binder连接池，可以为多个应用提供服务

第三章 View
  
1.View是一种界面层的控件的一种抽象，它代表了一个控件，ViewGroup包含了一组view，但同时也继承于View。
  
2.VelocityTracker 速度追踪，用于追踪手指在滑动中的速度 （代码示例 p126）
  
3.GestureDetector 手势检测，用于辅助检测用户的单击、滑动、长按、双击等功能。
  
4.Scroller ，弹性滑动对象，用于实现View的弹性滑动
  
5.View的滑动有三种方式：1.使用scrollTo/scrollBy 2.使用动画（View动画是对View的影响进行操作 p132） 3.改变布局参数
  
6.弹性滑动方式： 1.使用Scroller 2.通过动画 3.使用延时策略
  
7.View的事件分发机制：
  
用一段伪代码描述三个方法的关系：
  
public boolean dispatchTouchEvent(MotionEvent ev){
  
boolean consume = false;
  
if( onInterceptTouchEvent(ev) ){
  
consume = onTouchEvent(ev);
  
} else {
  
consume = child.dispatchTouchEvent(ev);
  
}
  
return consume;
  
}
  
8.拦截的一定是事件序列：不消耗ACTION\_DOWN，则事件序列都会由其父元素处理；只消耗ACTION\_DOWN事件，该事件会消失，消失的事件最终会交给Activity来处理；requestDisallowInterceptTouchEvent方法可以在子元素中干预父元素的事件分发过程，除了ACTION_DOWN。
  
9.((ViewGroup)getWindow().getDecorView().findViewById(android.R.id.content)).getChildAt(0); 可获得activity的contentview
  
10.View的滑动冲突有固定的解决规则。解决方式有：a.外部拦截法 b.内部拦截法
  
<!--more-->

第四章 View的工作原理

1.ViewRoot 对应于ViewRootImpl，它是连接WindowManager和DecorView的纽带，View的三大流程均是通过ViewRoot完成的。
  
2.View的绘制流程是从ViewRoot的performTraversals方法开始的，经过measure，layout，draw三个过程绘制出一个view。
  
3.DecorView作为顶级View,一般情况下它内部会包含一个竖直方向的linearLayout，这个LinearLayout里面有上下两个部分，上面为标题栏，下面是任务栏。
  
4.系统内部是通过MeasureSpec来进行View的测量，但是正常情况下我们使用View指定MeasureSpec，我们可以给View设置LayoutParams。在View测量的时候，系统会将LayoutParams在父容器的约束下转换成对应的MeasureSpec，然后再根据这个MeasureSpec来确定View测量后的宽高。
  
5.对于DecorView，其MeasureSpec由窗口的尺寸和自身的LayoutParams来共同确定，对于普通的View，MeasureSpec由父容器的MeasureSpec和自身的LayoutParams来共同决定，MeasureSpec一旦确定后，onMeasure中就可以确定View的测量宽高了
  
6.在某些极端情况下，系统可能需要多次measure才能确定最终的测量宽/高，在这种情形下，在onMeasure方法中拿到的测量可能是不准确的，一个比较好的习惯是在onLayout方法中去获取。
  
7.在onCreate,onStart,onResume中均无法正确得到某个View的宽高信息，提供四种方法解决：1.onWindowFocusChanged 2.view.post(runnable) 3.ViewTreeObserver 4.view.measure (分情况获取)
  
8.当我们的自定义控件继承ViewGroup并且本身不具备绘制功能时，可以开启setWillNotDraw这个方法从而便于系统进行后续优化
  
9.自定义View需知：
  
a.让View支持wrap_content
  
b.如果有必要，让你的View支持padding
  
c.尽量不要在view中使用handler，没必要
  
d.View中如果有线程或者动画，需要及时停止，参考View#onDetachedFromWindow
  
e.View带有滑动嵌套情形时，需要处理好滑动冲突

第五章 理解RemoteViews

1.RemoteViews标示的是一个View结构，它可以在其他进程中显示，主要使用场景有两种：通知栏和桌面小部件！
  
2.更新Remoteviews时，无法直接访问里面的View，必须通过它所提供的一系列方法，比如setTextViewText(),setImageViewResource().
  
3.RemoteView目前不能支持所有的View类型，所有支持类型不一一列举，请自行百度。
  
4.RemoteViews内部有一个mActions成员，它是一个ArrayList，外界每一次的操作方法都会生成一个Action并加入到这个list中，在apply方法中运行更新界面的代码

第六章 Android的Drawable

1.Drawable表示的是一种可以在Canvas上进行绘制的抽象的概念。
  
2.Drawable的工作原理很简答，其核心就是draw方法，可以通过重写Drawable的draw方法来自定义Drawable。

第七章 Android 动画深入分析

1.Android的动画可以分为三种：View动画，帧动画和属性动画
  
2.LayoutAnimation作用于ViewGroup，为ViewGroup指定一个动画，这样当它的子元素出场时都会具有这种动画效果。
  
3.TimeInterpolator 为时间插值器，它的作用是根据时间流逝的百分比来计算当前属性值改变的百分比，系统预支的有LinearInterpolator(线性插值器、匀速动画)、AccelerateDecelerateInterpolator(加速减速插值器：动画两头慢中间快)和DeceletateInterpolator（减速插值器：动画越来越慢）等。
  
4.View动画只支持四种类型：平移，旋转，缩放，不透明度。缩放只是把View放大了而已，字体背景都会被拉伸。
  
5.对object的属性abc做动画，如果想要让动画生效，必须满足两个条件：
       
a. object必须要提供setAbc方法，如果动画的时候没有传递初始值，那么还要提供getAbc方法，因为系统要去读取abc属性的初始值
       
b. object的setAbc对属性abc所做的改变必须能够通过某种方法反映出来,比如会带来UI的改变之类的

第八章 理解Window和WindowManager

1.android中所有的视图都是通过Window来呈现的，不管是activty，Dialog还是Toast，他们的试图实际上都是附加在Window上的，因此Window实际是View的直接管理者。
  
2.Window有三种类型，非别是应用Window，子Window和系统Window。
  
3.Activity 的Window创建过程：
       
a.如果没有DecorView，就创建它
       
b.将View添加到DecorView的mContentParent中
       
c.回调Activity的onContentChanged方法通知Activity视图已经发生改变
  
4.Dialog的Window创建过程
       
a.创建Window
       
b.初始化DecorView并将Dialog的试图添加到DecorView中
       
c.将DecorView添加到Window中并显示

第九章 四大组件的工作过程

1.Activity的启动过程，有些繁琐，不再赘述
  
2.Service的启动过程，绑定过程
  
3.ContentProvider所在的进程启动时,ContentProvider会同时启动并被发布到AMS中,需要注意的是，这个时候ContentProvider的onCreate要先于Application的onCreate而执行。
  
4.BroadcastReceiver的生命周期十分短暂，每次广播到来时,会重新创建BroadcastReceiver对象,并且调用onReceive()方法,执行完以后,该对象即被销毁.当onReceive()方法在10秒内没有执行完毕，Android会认为该程序无响应.所以在BroadcastReceiver里不能做一些比较耗时的操作,否侧会弹出ANR(Application NoResponse)的对话框。如果需要完成一项比较耗时的工作,应该通过发送Intent给Service,由Service来完成.这里不能使用子线程来解决,因为BroadcastReceiver的生命周期很短,子线程可能还没有结束BroadcastReceiver就先结束了.BroadcastReceiver一旦结束,此时BroadcastReceiver的所在进程很容易在系统需要内存时被优先杀死,因为它属于空进程(没有任何活动组件的进程).如果它的宿主进程被杀死,那么正在工作的子线程也会被杀死.所以采用子线程来解决是不可靠的.
  
5.（引用 http://www.cnblogs.com/lwbqqyumidi/p/4168017.html）
  
静态注册的广播接收器即使app已经退出，主要有相应的广播发出，依然可以接收到，但此种描述自Android 3.1开始有可能不再成立“
  
Android 3.1开始系统在Intent与广播相关的flag增加了参数，分别是FLAG\_INCLUDE\_STOPPED\_PACKAGES和FLAG\_EXCLUDE\_STOPPED\_PACKAGES。
  
FLAG\_INCLUDE\_STOPPED_PACKAGES：包含已经停止的包（停止：即包所在的进程已经退出）
  
FLAG\_EXCLUDE\_STOPPED_PACKAGES：不包含已经停止的包

主要原因如下：
  
自Android3.1开始，系统本身则增加了对所有app当前是否处于运行状态的跟踪。在发送广播时，不管是什么广播类型，系统默认直接增加了值为FLAG\_EXCLUDE\_STOPPED_PACKAGES的flag，导致即使是静态注册的广播接收器，对于其所在进程已经退出的app，同样无法接收到广播。
  
详情参加Android官方文档：http://developer.android.com/about/versions/android-3.1.html#launchcontrols
  
由此，对于系统广播，由于是系统内部直接发出，无法更改此intent flag值，因此，3.1开始对于静态注册的接收系统广播的BroadcastReceiver，如果App进程已经退出，将不能接收到广播。
  
但是对于自定义的广播，可以通过复写此flag为FLAG\_INCLUDE\_STOPPED_PACKAGES，使得静态注册的BroadcastReceiver，即使所在App进程已经退出，也能能接收到广播，并会启动应用进程，但此时的BroadcastReceiver是重新新建的。

1 Intent intent = new Intent();
  
2 intent.setAction(BROADCAST_ACTION);
  
3 intent.addFlags(Intent.FLAG\_INCLUDE\_STOPPED_PACKAGES);
  
4 intent.putExtra(&#8220;name&#8221;, &#8220;qqyumidi&#8221;);
  
5 sendBroadcast(intent);

注1：对于动态注册类型的BroadcastReceiver，由于此注册和取消注册实在其他组件（如Activity）中进行，因此，不受此改变影响。
  
注2：在3.1以前，相信不少app可能通过静态注册方式监听各种系统广播，以此进行一些业务上的处理（如即时app已经退出，仍然能接收到，可以启动service等..）,3.1后，静态注册接受广播方式的改变，将直接导致此类方案不再可行。于是，通过将Service与App本身设置成不同的进程已经成为实现此类需求的可行替代方案。

第十章 Android的消息机制

1.TheadLocal是一个线程内部的数据存储类，通过它可以在指定的线程中存储数据，数据存储以后，只有在指定线程中可以获取到存储的数据，对于其他线程来说则无法获取数据。
  
2.Activity的主线程就是ActivityThread，主线程的入口方法为main，在main方法中系统会通过Looper.prepareMainLooper()来创建主线程的Lopper以及MessageQueue，并通过Looper.loop()来开启主线程的消息循环。

第十一章 Android的线程和线程池

1.尽管AsyncTask、IntentService以及HandlerThread的表现形式都有别于传统的线程，但是他们的本质仍然是传统线程。对于AsyncTask来说，它的底层用到了线程池，对于IntentService和HandlerThread来说，它们的底层则直接使用了线程。
  
2.IntentService是一种服务，它不容易被系统杀死从而可以尽量保证任务的执行，而如果是一个后台线程，由于这个时候进程中没有活动的四大组件，那么这个进程的优先级会非常低，很容易被系统杀死，这就是IntentService的优点。
  
3.主线程是指进程所拥有的线程，在java中默认情况下一个进程只有一个线程，这个线程就是主线程。
  
4.AsyncTask提供了四个核心方法：
       
a.onPreExecute( ),在主线程中执行，在异步任务执行之前，此方法都会被调用，一般可以用于做一些准备工作。
       
b.doInBackground(Params&#8230;params)，在线程池中执行，用于执行异步任务。此方法中可以通过publishProgress方法来更新任务进度，publishProgress方法会调用onProgressUpdate方法，另外此方法还要返回计算的结果给onPostExecute方法。
       
c.onProgressUpdatre(Progress&#8230;valuses),在主线程中执行，当后台任务的执行进度发生改变时此方法会被调用。
       
d.onPostExecute(Result result),在主线程中执行，在一部人物执行之后，此方法会被调用，其中result参数是后台任务的返回值，即doInBackground的返回值。
  
5.AsyncTask使用注意项：
       
a.必须在主线程中加载。
       
b.AsyncTask的对象必须在主线程中创建。
       
c.execute方法必须在UI线程调用。
       
d.不要在程序中直接调用onPreExecute(),onPostExecute,doInBackground和onProgressUpdate方法。
       
e.一个AsyncTask对象只能执行一次，即只能调用一次execute方法，否则会报运行时异常
       
f.在Android1.6之前，AsyncTask是串行执行任务的，Android1.6的时候开始采用线程池处理并行任务，但是从Android3.0开始后，为了避免AsyncTask带来的并发错误，又采用一个线程来串行执行任务。尽管如此，我们可以通过AsyncTask的executeOnExecutor方法来并行的执行任务。
  
6.HandlerThread的run方法是一个无限循环，因此当明确不需要再使用时，可以通过它的quit或者quitSafely方法来终止线程的执行。
  
7.线程池的优点：
       
a.重用线程池中的线程，避免因为线程的创建和销毁所带来的性能开销。
       
b.能有效控制线程池的最大并发数，避免大量的线程志坚因互相抢占系统资源而导致的阻塞现象
       
c.能够对线程进行简单的管理，并提供定时执行以及指定间隔循环执行等功能
  
8.Android中常用的四种线程池：
       
a.FixedThreadPool,线程数量固定的线程池，当线程处于空闲状态时，他们并不会回收，除非线程池被关闭了
       
b.CachedThreadPool，线程数量不定的线程池，只有非核心线程，且最大线程数为Integer.MAX_VALUE
       
c.ScheduledThreadPool，核心线程数量是固定的，而非核心线程数量没有限制
       
d.SingleThreadExecutor，只有一个核心线程，确保所有的任务都在同一个线程中按顺序执行

第十二章 Bitmap的加载和Cache

1.采用BitmapFactory.Options来加载所需尺寸的图片，高效地加载bitmap
  
2.Android中的缓存策略：1.LruCache,内存缓存 2.DiskLruCache，磁盘缓存
  
3.优化列表的卡顿现象：
       
a.不要在getView中执行耗时操作
       
b.控制异步任务的执行频率
       
c.开启硬件加速

第十三章 综合技术

1.使用CrashHandler来获取应用的crash信息
  
2.使用multidex来解决方法数越界
  
3.Android动态加载技术在项目中的使用

第十四章 JNI和NDK编程

1.使用NDK有如下好处：
       
a.提高代码的安全性
       
b.可以很方便地使用目前已有的C/C++开源库
       
c.便于平台间的移植
       
d.提高程序在某些特定情形下的执行效率

第十五章 Android性能优化

1.布局优化：布局优化的思想很简单，就是尽量减少布局文件的层级。
       
a.删除不居中无用的控件和层级
       
b.有选择的使用性能较低的ViewGroup，比如RelativeLayout.如果布局中可以使用LinearLayout也可以使用RelativeLayout，那么就采用LinearLayout.
       
c.采用<include>，<merge>和Viewstub.
  
2.绘制优化：
       
a.onDraw中不要创建新的局部对象，这是因为onDraw方法可能会被频繁调用
       
b.onDraw方法中不要做耗时的任务
  
3.内存泄露优化：
       
a.静态变量导致的内存泄露
       
b.单例模式导致的内存泄露
       
c.属性动画导致的内存泄露
  
4.ListView和Bitmap优化
  
5.线程优化
  
6.使用MAT工具分析内存泄露
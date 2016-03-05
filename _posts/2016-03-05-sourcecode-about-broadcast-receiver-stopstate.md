---
layout:     post
title:      "Android设置中“强行停止”详解"
subtitle:   "应用停止状态源码解析"
date: 2016-03-05T12:00:19+00:00
author:     "Roger"
header-img: "img/home-bg-o.jpg"
tags:
    - Android Framework
---
最近工作上遇到了广播接受不到的问题，查看了《Android 开发艺术探索》一书中关于广播的发送和接受的章节（P356-P362）。其中（P358）介绍了从Android 3.1 之后广播的一些区别 。

从 Android 3.1 开始，系统为所有的广播都默认添加了FLAG_EXCLUDE_STOPPED_PACKAGES 标志。所有处于停止状态的应用将无法接受到该标志的广播。注意，只有两种情况下应用才会处于停止状态：

   1. 应用安装后未运行
   2. 应用被手动（设置-应用-强制停止）或者其他应用强制停止了

如果需要启动处于停止状态的应用，则只要为Intent添加 FLAG_INCLUDE_STOPPED_PACKAGES 标记即可。

详细请查看 <a href="http://droidyue.com/blog/2014/07/14/look-inside-android-package-stop-state-since-honeycomb-mr1/index.html">你造么,Android中程序的停止状态</a>

技术小黑屋的这个解析的确很完整，但是在问答环节我看到这样一个问题：

* 提问:如果我的程序没有activity只有一个receiver,我改如何激活才能接收到正常的广播intent呢

* 回答:实际上,如果是上面所述的情况,该应用在安装之后不是处于停止状态,因为它没有任何用户可以直接点击的行为去将它移除停止状态.你可以正常接收广播intent,除非你人为地将它强制停止.

在页面的最底端我也看到了有人这样留言：

 * 你好，我写了一个只有receiver和service的app，安装后是无法接受系统广播的，后来用了一个自定义的广播启动它过一次后，才开始接受系统广播了

 * 我认为你的那个app安装之后不是处于停止状态（关于验证，你可以查看系统设置中的应用程序，那里面能准确的看出是否为停止状态），另外我感觉你应该确认一下系统广告是否发出，因为你后面的说，自定义的广告可以接收即表明这个程序处于非停止状态。希望可以确认一下哈。

看到这个回答，我的内心是抗拒的。于是我自己写了个Demo测试了一下，非常简单的代码：

AndroidManifest.xml：


```xml
   <receiver
	   android:name="com.roger.BroadcastTest.Receiver"
	   android:enabled="true"
	   android:exported="true" >
	   <intent-filter>
	       <action android:name="android.intent.action.BOOT_COMPLETED" />
	       <action android:name="com.roger.broadcast.test" />
	   </intent-filter>
	</receiver> 
```

	
JAVA:

	public class Receiver extends BroadcastReceiver {
	
		@Override
		public void onReceive(Context arg0, Intent arg1) {
			// TODO Auto-generated method stub
			Log.i("Tag", "receive:" + arg1.getAction());
		}
	}

应用中就注册了一个广播接收器，安装后直接重启，果然并没有收到开机成功的广播。然而设置中查看，该应用并没有处于停止状态。所以我猜想，安装后未启动的应用（未启动过该应用中任何四大组件），虽然在设置中显示未停止，但在framework中，应该是处于停止状态的。

来，开始苦逼的看代码之路，我想要搞清楚的有两点：

1. framework中广播分发者如何区分广播接收器是否处于停止状态

2. framework中如何将应用从停止状态切换到非停止状态

开始第一点，对于广播分发的过程主要参考，请参考博客： <a href="http://my.oschina.net/youranhongcha/blog/226274#OSC_h3_7">品茗论道说广播(Broadcast内部机制讲解)</a> ，介绍的很详细。

我们从Intent中开始，介绍一个framework源码搜索网站：<a href="http://androidxref.com/">http://androidxref.com/</a>,我们选4.4的源码，搜索FLAG_INCLUDE_STOPPED_PACKAGES，点击链接跟进到Intent源码中，看到这样一个方法：


	4964    /** @hide */
	4965    public boolean isExcludingStopped() {
	4966        return (mFlags&(FLAG_EXCLUDE_STOPPED_PACKAGES|FLAG_INCLUDE_STOPPED_PACKAGES))
	4967                == FLAG_EXCLUDE_STOPPED_PACKAGES;
	4968    }


很明显，这个方法返回的是发送的广播是否需要包含处于停止状态的应用。再跟进改方法的使用者，跟进到IntentResolver.java中：


	523        final boolean excludingStopped = intent.isExcludingStopped();


这个变量在哪使用呢？


	543            if (excludingStopped && isFilterStopped(filter, userId)) {
	544                if (debug) {
	545                    Slog.v(TAG, "  Filter's target is stopped; skipping");
	546                }
	547                continue;
	548            }


在buildResoleList中，Good,看到log我们似乎离成功不远了，那么这个isFilterStopped方法肯定是判断应用是否处于停止状态的，再更进：


	332    /**
	333     * Returns whether the object associated with the given filter is
	334     * "stopped," that is whether it should not be included in the result
	335     * if the intent requests to excluded stopped objects.
	336     */
	337    protected boolean isFilterStopped(F filter, int userId) {
	338        return false;
	339    }


什么鬼~？直接返回false？逗我吗？..再认真看下注释，大概意思是传到这个方法的filter必须已经排除了'stopped' 的应用，这？应该是被重写了，继续在全局搜索该方法名，果然，在PackageManagerService中，看到了如下代码：


	5710        @Override
	5711        protected boolean isFilterStopped(PackageParser.ActivityIntentInfo filter, int userId) {
	5712            if (!sUserManager.exists(userId)) return true;
	5713            PackageParser.Package p = filter.activity.owner;
	5714            if (p != null) {
	5715                PackageSetting ps = (PackageSetting)p.mExtras;
	5716                if (ps != null) {
	5717                    // System apps are never considered stopped for purposes of
	5718                    // filtering, because there may be no way for the user to
	5719                    // actually re-launch them.
	5720                    return (ps.pkgFlags&ApplicationInfo.FLAG_SYSTEM) == 0
	5721                            && ps.getStopped(userId);
	5722                }
	5723            }
	5724            return false;
	5725        }


这下无误了吧！再看到380行：


	375    // All available activities, for your resolving pleasure.
	376    final ActivityIntentResolver mActivities =
	377            new ActivityIntentResolver();
	378
	379    // All available receivers, for your resolving pleasure.
	380    final ActivityIntentResolver mReceivers =
	381            new ActivityIntentResolver();
	382
	383    // All available services, for your resolving pleasure.
	384    final ServiceIntentResolver mServices = new ServiceIntentResolver();
	385
	386    // All available providers, for your resolving pleasure.
	387    final ProviderIntentResolver mProviders = new ProviderIntentResolver();


379行注释：所有有效的receivers都被这个ActivityIntentResolver解析，Good！现在我们看到主要是通过PackageSetting这个类中的getStopped方法来判断应用是否处于停止状态。那么有哪些动作将会对这个值有影响呢？看到PackageSettingBase中关于设定改值的方法：


	251    void setStopped(boolean stop, int userId) {
	252        modifyUserState(userId).stopped = stop;
	253    }


再通过查询调用者，我们会发现主要有以下代码将影响改值：

1. 从停止到非停止状态（成功启动该应用中的四大组件即可）
  * ActivityStack中
      
		      1426        // Launching this app's activity, make sure the app is no longer
		      1427        // considered stopped.
		      1428        try {
		      1429            AppGlobals.getPackageManager().setPackageStoppedState(
		      1430                    next.packageName, false, next.userId); /* TODO: Verify if correct userid */
		      1431        }
      
  * ActiveServices中
      
		      1223        // Service is now being launched, its package can't be stopped.
		      1224        try {
		      1225            AppGlobals.getPackageManager().setPackageStoppedState(
		      1226                    r.packageName, false, r.userId);
		      1227        }
      
  * ActivityManagerService中
      
		      7548                        // Content provider is now in use, its package can't be stopped.
		      7549                        try {
		      7550                            AppGlobals.getPackageManager().setPackageStoppedState(
		      7551                                    cpr.appInfo.packageName, false, userId);
		      7552                        }
      
  * BroadcastQueue中
      
		      841            // Broadcast is being executed, its package can't be stopped.
		      842            try {
		      843                AppGlobals.getPackageManager().setPackageStoppedState(
		      844                        r.curComponent.getPackageName(), false, UserHandle.getUserId(r.callingUid));
		      845            }
      
2. 从非停止状态到停止状态（初始化以及设置中点击强制停止）
  * 在/frameworks/base/services/java/com/android/server/pm/Settings.java 中 有对 "packages-stopped.xml"文件的读操作，未找到写操作，但我相信在应用安装成功后会加入"packages-stopped.xml"文件中
  
  * 设置中点击强制停止 /packages/apps/Settings/src/com/android/settings/applications/ProcessStatsDetail.java：
        
	        279    private void killProcesses() {
	        280        ActivityManager am = (ActivityManager)getActivity().getSystemService(
	        281                Context.ACTIVITY_SERVICE);
	        282        am.forceStopPackage(mEntry.mUiPackage);
	        283        checkForceStop();
	        284    }
        

至此，通过源码我们已经了解，如果新安装的应用，未曾成功启动过四大组件，默认是处于停止状态的，这也是Google对系统的保护想要达到的效果。

That's all , wish you have a good weekend.

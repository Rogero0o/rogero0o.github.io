---
layout:     post
title:      "android:process 的坑，你懂吗？"
subtitle:   "android:process 属性详解及注意事项"
date: 2016-03-17 19:12:01 +0800
author:     "Roger"
header-img: "img/home-bg-o.jpg"
tags:
    - Android Framework
---
android:process 的坑，你懂吗？
=============


许多知识知其然而不知其所以然，这也许就是大神与菜鸟的区别吧。

最近排查问题时发现一个问题： 一个在 Application 中启动的定时任务在运行时会被调用多次，诡异的很，最后发现是一个前人留下的坑，原因就是对 android:process 不知其所以然造成的。


### android:process 属性

关于 android:process 属性，相信大家都不陌生，android 官网是这样说明的 ：

>默认情况下，同一应用的所有组件均在相同的进程中运行，且大多数应用都不会改变这一点。 但是，如果您发现需要控制某个组件所属的进程，则可在清单文件中执行此操作。

>各类组件元素的清单文件条目—&lt;activity>、&lt;service>、&lt;receiver> 和 &lt;provider>—均支持 android:process 属性，此属性可以指定该组件应在哪个进程运行。您可以设置此属性，使每个组件均在各自的进程中运行，或者使一些组件共享一个进程，而其他组件则不共享。 此外，您还可以设置 android:process，使不同应用的组件在相同的进程中运行，但前提是这些应用共享相同的 Linux 用户 ID 并使用相同的证书进行签署。

>此外，<application> 元素还支持 android:process 属性，以设置适用于所有组件的默认值。

>如果内存不足，而其他为用户提供更紧急服务的进程又需要内存时，Android 可能会决定在某一时刻关闭某一进程。在被终止进程中运行的应用组件也会随之销毁。 当这些组件需要再次运行时，系统将为它们重启进程。

>决定终止哪个进程时，Android 系统将权衡它们对用户的相对重要程度。例如，相对于托管可见 Activity 的进程而言，它更有可能关闭托管屏幕上不再可见的 Activity 进程。 因此，是否终止某个进程的决定取决于该进程中所运行组件的状态。 下面，我们介绍决定终止进程所用的规则。

在需要使用到新进程时，可以使用 android:process 属性，如果被设置的进程名是以一个冒号开头的，则这个新的进程对于这个应用来说是私有的，当它被需要或者这个服务需要在新进程中运行的时候，这个新进程将会被创建。如果这个进程的名字是以字符开头，并且符合 android 包名规范(如 com.roger 等)，则这个服务将运行在一个以这个名字命名的全局的进程中，当然前提是它有相应的权限。若以数字开头(如 1Remote.com )，或不符合 android 包名规范（如 Remote），则在编译时将会报错 （ INSTALL_PARSE_FAILED_MANIFEST_MALFORMED ）。新建进程将允许在不同应用中的各种组件可以共享一个进程，从而减少资源的占用。具体可以参考博客：[apk，task，android:process与android:sharedUserId的区别](http://blog.csdn.net/lynn0708/article/details/13624403)

重点来了，因为设置了 android:process 属性将组件运行到另一个进程，相当于另一个应用程序，所以在另一个线程中也将新建一个 Application 的实例。**因此，每新建一个进程   Application 的 onCreate 都将被调用一次。** 如果在 Application 的 onCreate 中有许多初始化工作并且需要根据进程来区分的，那就需要特别注意了。

让我们到 Framework 中看看新建进程的逻辑，请打开老罗的博客 ： [Android系统在新进程中启动自定义服务过程（startService）的原理分析](http://blog.csdn.net/luoshengyang/article/details/6677029)

详细介绍了新进程启动的过程，其中我们重点看到 `Step 17. ActivityThread.handleCreateService` 中

	public final class ActivityThread {  

	    ......  

	    private final void handleCreateService(CreateServiceData data) {  
	        // If we are getting ready to gc after going to the background, well  
	        // we are back active so skip it.  
	        unscheduleGcIdler();  

	        LoadedApk packageInfo = getPackageInfoNoCheck(  
	            data.info.applicationInfo);  
	        Service service = null;  
	        try {  
	            java.lang.ClassLoader cl = packageInfo.getClassLoader();  
	            service = (Service) cl.loadClass(data.info.name).newInstance();  
	        } catch (Exception e) {  
	            if (!mInstrumentation.onException(service, e)) {  
	                throw new RuntimeException(  
	                    "Unable to instantiate service " + data.info.name  
	                    + ": " + e.toString(), e);  
	            }  
	        }  

	        try {  
	            if (localLOGV) Slog.v(TAG, "Creating service " + data.info.name);  

	            ContextImpl context = new ContextImpl();  
	            context.init(packageInfo, null, this);  

	            Application app = packageInfo.makeApplication(false, mInstrumentation);  
	            context.setOuterContext(service);  
	            service.attach(context, this, data.info.name, data.token, app,  
	                ActivityManagerNative.getDefault());  
	            service.onCreate();  
	            mServices.put(data.token, service);  
	            try {  
	                ActivityManagerNative.getDefault().serviceDoneExecuting(  
	                    data.token, 0, 0, 0);  
	            } catch (RemoteException e) {  
	                // nothing to do.  
	            }  

	        } catch (Exception e) {  
	            if (!mInstrumentation.onException(service, e)) {  
	                throw new RuntimeException(  
	                    "Unable to create service " + data.info.name  
	                   	 + ": " + e.toString(), e);  
	            }  
	        }  
	    }  

	    ......  

	}  

看到这行 `Application app = packageInfo.makeApplication(false, mInstrumentation);` 在这里创建了 Application 。

### 解决方案

获取当前运行进程的名称：

#### 方案1

	public static String getProcessName(Context cxt, int pid) {  
        ActivityManager am = (ActivityManager) cxt.getSystemService(Context.ACTIVITY_SERVICE);  
        List<RunningAppProcessInfo> runningApps = am.getRunningAppProcesses();  
        if (runningApps == null) {  
            return null;  
        }  
        for (RunningAppProcessInfo procInfo : runningApps) {  
            if (procInfo.pid == pid) {  
                return procInfo.processName;  
            }  
        }  
        return null;  
    }  

目前网上主流的方法，但效率没有方案2高，感谢由王燚同学提供的方案2

#### 方案2

    public static String getProcessName() {
      try {
        File file = new File("/proc/" + android.os.Process.myPid() + "/" + "cmdline");
        BufferedReader mBufferedReader = new BufferedReader(new FileReader(file));
        String processName = mBufferedReader.readLine().trim();
        mBufferedReader.close();
        return processName;
      } catch (Exception e) {
        e.printStackTrace();
        return null;
      }
    }

然后在 Application 的 onCreate 中获取进程名称并进行相应的判断，例如：

	String processName = getProcessName(this, android.os.Process.myPid());

	if (!TextUtils.isEmpty(processName) && processName.equals(this.getPackageName())) {//判断进程名，保证只有主进程运行
		//主进程初始化逻辑
		....
	}

### 总结

知其然还需知其所以然，这才是总结并提高的法宝。希望能帮到有需要的同学 ：）

Have a good day ~

### 参考

[http://blog.csdn.net/jason0539/article/details/45555671](http://blog.csdn.net/jason0539/article/details/45555671)

[Android系统在新进程中启动自定义服务过程（startService）的原理分析](http://blog.csdn.net/luoshengyang/article/details/6677029)

[apk，task，android:process与android:sharedUserId的区别](http://blog.csdn.net/lynn0708/article/details/13624403)

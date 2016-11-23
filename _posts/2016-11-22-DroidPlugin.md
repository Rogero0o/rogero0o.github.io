---
layout:     post
title:      "Android 插件化框架 DroidPlugin 学习笔记"
subtitle:   "Keep Learning"
date: 2016-11-22 11:00:01 +0800
author:     "Roger"
header-img: "img/home-bg-o.jpg"
tags:
    - Android Framework
---
Android 插件化框架 *DroidPlugin* 学习笔记
---

上一篇我们对 DL 框架的思路进行了一些总结，总的来说就是通过一个代理的 *activity* 作为傀儡来控制插件 *activity* 的生命周期，通过 *AssetManager* 的隐藏方法 *addAssetPath* 来解决加载资源的问题。但是同时，DL 框架存在的缺点就是比较依赖 *that* 语法，开发插件程序和主程序的代码需要单独区分。在这两点问题上，360 助手的插件化框架 *DroidPlugin* 似乎解决的更好一些，这个框架基本 *Hook* 了系统所有的 *Service* ,欺骗了系统大部分的 *API* ,编写插件程序和开发普通 *app* 没有任何区别，这是 *DroidPlugin* 框架的优点，同时，由于需要 *Hook* 大量系统 *API*，所以 *DroidPlugin* 在机型适配上存在的问题就比较严重，国内各个厂商的 *Rom* 可能都需要进行手动适配，这是 *DroidPlugin* 最致命的缺点，其次就是因为 *Hook* 必须是基于反射的，所以在性能上会有部分损耗。有一些大厂（例如酷狗）正是因为 *DroidPlugin* 在适配上的短板所以选择了 *DL* 作为插件化框架，所以这也需要使用者自己进行斟酌。

*DroidPlugin* 的源码解析博客：[Link](http://weishu.me/2016/01/28/understand-plugin-framework-overview/)

首先学习一下 *DroidPlugin* 的基本原理，使用代理模式进行 *API Hook* 进而达到在 *Framework* 中近乎于随心所欲的添加可运行代码。代理模式基本类似于以下实现：

    interface RentRoom {
        public void rent();
    }


    class People implements RentRoom {

        @Override
        public void rent() {
          // TODO Auto-generated method stub
          System.out.println("我是个人，我要租房");
        }

    }

    class ProxyRent implements RentRoom {

  		  private People mPeople;


    		public ProxyRent(People mPeople) {
      			super();
      			this.mPeople = mPeople;
    		}

    		@Override
    		public void rent() {
      			// TODO Auto-generated method stub
      			System.out.println("我是中介，我在people租房前运行");
      			mPeople.rent();
      			System.out.println("我是中介，我在people租房后运行");
    		}
  	}    

    public static void main(String[] args) {
    		People mPeople = new People();
    		ProxyRent mProxyRent = new ProxyRent(mPeople);
    		mProxyRent.rent();
  	}

我们有一个 *RentRoom* 接口，里面有一个 *rent()* 方法表示租房子，有一个 *People* 类，说明了人有租房的需要，当然我们可以亲自去租房，那么直接调用 *mPeople* 的 *rent()* 方法，但是我们可能有别的事情要忙，于是就有了 *ProxyRent* 这个租房的中介，于是我们只要将 *People* 介绍给中介类，直接让中介去租房就可以了，但是这时候，中介在租房的时候就可以做一些事情，如上面例子中就是打印了一些日志，当然还可以做更多的事情，比如我们只需要建立一个代理类，将系统源码里的变量通过反射替换成我们创建的代理类，那么我们即可以在代理类中实现自己的逻辑，比如启动我们自己的 *activity* 等等，这就是 *DroidPlugin* 的 *API Hook* 的最基本的原理，下面还是使用原文博客中的例子 *Hook* 掉 *startActivity* 方法，使得每次调用的时候都打印一条日志。

这里还需要注意一个 *Hook* 点的把握，在 *Hook* 对象的选择上，只有 **容易找到的对象** 比较容易被 *Hook* ，那么什么又是比较容易找到的对象呢？**静态变量和单例** ，作者在博客中介绍到：“在一个进程之内，静态变量和单例变量是相对不容易发生变化的，因此非常容易定位，而普通的对象则要么无法标志，要么容易改变”。

看到 *ConetxtImpl* 类的 *startActivity* 方法：

    @Override
    public void startActivity(Intent intent, Bundle options) {
        warnIfCallingFromSystemProcess();
        if ((intent.getFlags()&Intent.FLAG_ACTIVITY_NEW_TASK) == 0) {
            throw new AndroidRuntimeException(
                    "Calling startActivity() from outside of an Activity "
                    + " context requires the FLAG_ACTIVITY_NEW_TASK flag."
                    + " Is this really what you want?");
        }
        mMainThread.getInstrumentation().execStartActivity(
            getOuterContext(), mMainThread.getApplicationThread(), null,
            (Activity)null, intent, -1, options);
    }

可以看到这里最终调用的是 *mMainThread.getInstrumentation().execStartActivity* ，而 *mMainThread.getInstrumentation()* 返回的是一个 *mInstrumentation* 成员变量，而 *mMainThread* 是主线程，所以它只有一个，因此这里是一个很好的 *Hook* 点，我们需要做的就是将 *mMainThread* 的 *mInstrumentation* *Hook* 成我们自己实现的代理类，就能在每次启动的时候加入我们自己的逻辑，打印一条日志。

首先我们需要拿到 *mMainThread* 这个对象，可以通过调用 *ActivityThread* 的一个方法 *currentActivityThread* 来获取，而 *ActivityThread* 是一个隐藏类，所以需要用到反射：

    // 先获取到当前的ActivityThread对象
    Class<?> activityThreadClass = Class.forName("android.app.ActivityThread");
    Method currentActivityThreadMethod = activityThreadClass.getDeclaredMethod("currentActivityThread");
    currentActivityThreadMethod.setAccessible(true);
    Object currentActivityThread = currentActivityThreadMethod.invoke(null);

拿到 *currentActivityThread* 这个对象之后我们需要做的就是将这个类中的 *mInstrumentation* 成员变量替换成我们创建的代理类，所以首先需要编写一个 *mInstrumentation* 的代理类：

    public class EvilInstrumentation extends Instrumentation {

        private static final String TAG = "EvilInstrumentation";

        // ActivityThread中原始的对象, 保存起来
        Instrumentation mBase;

        public EvilInstrumentation(Instrumentation base) {
            mBase = base;
        }

        public ActivityResult execStartActivity(
                Context who, IBinder contextThread, IBinder token, Activity target,
                Intent intent, int requestCode, Bundle options) {

            // Hook之前, XXX到此一游!
            Log.d(TAG, "\n执行了startActivity, 参数如下: \n" + "who = [" + who + "], " +
                    "\ncontextThread = [" + contextThread + "], \ntoken = [" + token + "], " +
                    "\ntarget = [" + target + "], \nintent = [" + intent +
                    "], \nrequestCode = [" + requestCode + "], \noptions = [" + options + "]");

            // 开始调用原始的方法, 调不调用随你,但是不调用的话, 所有的startActivity都失效了.
            // 由于这个方法是隐藏的,因此需要使用反射调用;首先找到这个方法
            try {
                Method execStartActivity = Instrumentation.class.getDeclaredMethod(
                        "execStartActivity",
                        Context.class, IBinder.class, IBinder.class, Activity.class,
                        Intent.class, int.class, Bundle.class);
                execStartActivity.setAccessible(true);
                return (ActivityResult) execStartActivity.invoke(mBase, who,
                        contextThread, token, target, intent, requestCode, options);
            } catch (Exception e) {
                // 某该死的rom修改了  需要手动适配
                throw new RuntimeException("do not support!!! pls adapt it");
            }
        }
    }

我们看到 *EvilInstrumentation* 这个类继承了 *Instrumentation* ，并且重写了 *execStartActivity* 这个方法，打印了我们需要的日志，并且调用了原变量的 *execStartActivity* 方法，保证原功能的正常运行，下一步将 *currentActivityThread* 中的变量 *mInstrumentation* 替换掉即可：

    public static void attachContext() throws Exception {

        // 先获取到当前的ActivityThread对象
        Class<?> activityThreadClass = Class.forName("android.app.ActivityThread");
        Method currentActivityThreadMethod = activityThreadClass.getDeclaredMethod("currentActivityThread");
        currentActivityThreadMethod.setAccessible(true);
        Object currentActivityThread = currentActivityThreadMethod.invoke(null);

        // 拿到原始的 mInstrumentation字段
        Field mInstrumentationField = activityThreadClass.getDeclaredField("mInstrumentation");
        mInstrumentationField.setAccessible(true);
        Instrumentation mInstrumentation = (Instrumentation) mInstrumentationField.get(currentActivityThread);

        // 创建代理对象
        Instrumentation evilInstrumentation = new EvilInstrumentation(mInstrumentation);

        // 偷梁换柱
        mInstrumentationField.set(currentActivityThread, evilInstrumentation);
    }

通过反射，我们将原来的变量 *Hook* 成了我们的改造过的代理类，于是在代理类中即可以添加我们的代码来实现需要的功能需求~Woo，简直是打开了新世界！感谢原作者的分享 ：[Link](http://weishu.me/2016/01/28/understand-plugin-framework-overview/)

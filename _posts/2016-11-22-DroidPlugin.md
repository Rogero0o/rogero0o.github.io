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

    class ProxyRent implements RentRoom{


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

我们有一个 *RentRoom* 接口，里面有一个 *rent()* 方法表示租房子，有一个 *People* 类，说明了人有租房的需要，当然我们可以亲自去租房，那么直接调用 *mPeople* 的 *rent()* 方法，但是我们可能有别的事情要忙，于是就有了 *ProxyRent* 这个租房的中介，于是我们只要将 *People* 介绍给中介类，直接让中介去租房就可以了，但是这时候，中介在租房的时候就可以做一些事情，如上面例子中就是打印了一些日志，当然还可以做更多的事情，比如我们只需要建立一个代理类，将系统源码里的变量通过反射替换成我们创建的代理类，那么我们即可以在代理类中实现自己的逻辑，比如增加一些

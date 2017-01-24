---
layout:     post
title:      "AIDL与Binder机制学习笔记"
subtitle:   "The note about AIDL and Binder"
date: 2015-09-25T17:39:22+00:00
author:     "Roger"
header-img: "img/android-bg6.jpg"
tags:
    - Android Framework
---
最近学习了Binder机制内容，研究了好多大牛的博客，不过惭愧许多都看得云里雾里，最后通过不懈努力死缠烂打终于摸到一些门道，特此记录一下。

主要研究的博客：<a href="http://blog.csdn.net/lmj623565791/article/details/38461079" title="Android aidl Binder框架浅析" target="_blank">Android aidl Binder框架浅析</a> <a href="http://blog.csdn.net/luoshengyang/article/details/6642463" title="Android系统进程间通信Binder机制在应用程序框架层的Java接口源代码分析" target="_blank">Android系统进程间通信Binder机制在应用程序框架层的Java接口源代码分析</a>

本文使用的源码是第一篇博客大牛的源码，<a href="http://download.csdn.net/detail/lmj623565791/7767707" title="下载地址" target="_blank">下载地址</a>

在Android系统的Binder机制中，由一系统组件组成，分别是Client、Server、Service Manager和Binder驱动程序，其中Client、Server和Service Manager运行在用户空间，Binder驱动程序运行内核空间。Binder就是一种把这四个组件粘合在一起的粘结剂了，其中，核心组件便是Binder驱动程序了，Service Manager提供了辅助管理的功能，Client和Server正是在Binder驱动和Service Manager提供的基础设施上，进行Client-Server之间的通信。

从具体作用上来说：

服务器端接口：实际上是 Binder 类的对象，该对象一旦创建，内部则会启动一个隐藏线程，会接收Binder驱动发送的消息，收到消息后，会执行 Binder 对象中的 onTransact()函数，并按照该函数的参数执行不同的服务器端代码。

Binder驱动：该对象也为Binder类的实例，客户端通过该对象访问远程服务。

客户端接口：获得Binder驱动，调用其transact()发送消息至服务器

最简单的实现Binder机制的方法就是AIDL了，先分析一下AIDL中的Binder机制。

<!--more-->

代码可以参考源码中的 zhy\_binder\_aidl\_server02（服务端） 和 zhy\_binder_client02-tmp （客户端），运行结果请参考博客 <a href="http://blog.csdn.net/lmj623565791/article/details/38461079" title="Android aidl Binder框架浅析" target="_blank">Android aidl Binder框架浅析</a>

下面分析一下客户端通过AIDL调用服务端的过程。

首先，客户端通过bindService绑定服务端，用来获取服务端的binder接口：

	 private ICalcAIDL mCalcAidl;
	    private ServiceConnection mServiceConn = new ServiceConnection()
	    {
	        @Override
	        public void onServiceDisconnected(ComponentName name)
	        {
	            Log.e("client", "onServiceDisconnected");
	            mCalcAidl = null;
	        }

	        @Override
	        public void onServiceConnected(ComponentName name, IBinder service)
	        {
	            Log.e("client", "onServiceConnected");
	            mCalcAidl = ICalcAIDL.Stub.asInterface(service);
	        }
	    };

再看到如下 AIDL 中的代码：

		public static com.roger.aidl.mInterface asInterface(android.os.IBinder obj) {
			if ((obj == null)) {
				return null;
			}
			android.os.IInterface iin = obj.queryLocalInterface(DESCRIPTOR);
			if (((iin != null) && (iin instanceof com.roger.aidl.mInterface))) {
				return ((com.roger.aidl.mInterface) iin);
			}
			return new com.roger.aidl.mInterface.Stub.Proxy(obj);
		}

这里的obj是一个BinderProxy对象，它的queryLocalInterface返回null，于是调用下面语句获得服务端的的远程接口：

	return new com.zhy.calc.aidl.ICalcAIDL.Stub.Proxy(obj);

其实相当于调用了

	return new com.zhy.calc.aidl.ICalcAIDL.Stub.Proxy(new BinderProxy());

由于具体过程涉及到底层native的实现，具体过程请参考<a href="http://blog.csdn.net/luoshengyang/article/details/6642463" title="Android系统进程间通信Binder机制在应用程序框架层的Java接口源代码分析" target="_blank">Android系统进程间通信Binder机制在应用程序框架层的Java接口源代码分析</a>中的

四. Client获取HelloService的Java远程接口的过程

在客户端拿到服务端的远程接口之后，就可以开始跨进程通讯了。

看到点击加法调用的代码：

	 /**
	     * 点击12+12按钮时调用
	     * @param view
	     */
	    public void addInvoked(View view) throws Exception
	    {

	        if (mCalcAidl != null)
	        {
	            int addRes = mCalcAidl.add(12, 12);
	            Toast.makeText(this, addRes + "", Toast.LENGTH_SHORT).show();
	        } else
	        {
	            Toast.makeText(this, "服务器被异常杀死，请重新绑定服务端", Toast.LENGTH_SHORT)
	                    .show();

	        }

	    }

主要是 mCalcAidl.add(12, 12); ，我们继续跟进：

	@Override
	public int add(int x, int y) throws android.os.RemoteException
	{
	    android.os.Parcel _data = android.os.Parcel.obtain();
	    android.os.Parcel _reply = android.os.Parcel.obtain();
	    int _result;
	    try {
	        _data.writeInterfaceToken(DESCRIPTOR);
	        _data.writeInt(x);
	        _data.writeInt(y);
	        mRemote.transact(Stub.TRANSACTION_add, _data, _reply, );
	        _reply.readException();
	        _result = _reply.readInt();
	    }
	    finally {
	        _reply.recycle();
	        _data.recycle();
	        }
	    return _result;
	}

首先声明两个Parcel对象，一个用于传递数据，一个用户接收返回的数据

	_data.writeInterfaceToken(DESCRIPTOR);与服务器端的enforceInterfac对应
	_data.writeInt(x);
	_data.writeInt(y);写入需要传递的参数

调用了mRemote.transact方法，请注意这个mRemote正是在绑定服务成功时 onServiceConnected 方法返回的IBinder实例。

开始我们就说到 客户端的动作主要是获得Binder驱动，调用其transact()发送消息至服务器，具体代码实现就是这一步了。mRemote就是Binder驱动。

再深入进去请参考<a href="http://blog.csdn.net/luoshengyang/article/details/6642463" title="Android系统进程间通信Binder机制在应用程序框架层的Java接口源代码分析" target="_blank">Android系统进程间通信Binder机制在应用程序框架层的Java接口源代码分析</a>中的

五. Client通过HelloService的Java远程接口来使用HelloService提供的服务的过程

可以看到onTransact有四个参数

	code ， data ，replay ， flags

code 是一个整形的唯一标识，用于区分执行哪个方法，客户端会传递此参数，告诉服务端执行哪个方法

data客户端传递过来的参数

replay服务器返回回去的值

flags标明是否有返回值，0为有（双向），1为没有（单向）

	_reply.readException();

	\_result = \_reply.readInt();

最后读出我们服务端返回的数据，然后return。可以看到和服务端的onTransact基本是一模一样的。

综上所述，AIDL其实是以一套模板自动生成了调用Binder机制的java代码，其中客户端主要使用 asInterface 方法来获得服务端的Binder驱动，并通过驱动来调用服务端实现的 onTransact 方法，进而调用具体实现。

其实完全可以不用AIDL自己实现Binder机制，具体代码在 zhy\_binder\_aidl\_server02（服务端）和 zhy\_binder_client03 （客户端）中

由乘法为例：

客户端：

	public void mulInvoked(View view)
	    {

	        if (mPlusBinder == null)
	        {
	            Toast.makeText(this, "未连接服务端或服务端被异常杀死", Toast.LENGTH_SHORT).show();
	        } else
	        {
	            android.os.Parcel _data = android.os.Parcel.obtain();
	            android.os.Parcel _reply = android.os.Parcel.obtain();
	            int _result;
	            try
	            {
	                _data.writeInterfaceToken("CalcPlusService");
	                _data.writeInt(50);
	                _data.writeInt(12);
	                mPlusBinder.transact(0x110, _data, _reply, );
	                _reply.readException();
	                _result = _reply.readInt();
	                Toast.makeText(this, _result + "", Toast.LENGTH_SHORT).show();

	            } catch (RemoteException e)
	            {
	                e.printStackTrace();
	            } finally
	            {
	                _reply.recycle();
	                _data.recycle();
	            }
	        }

	    }

mPlusBinder 为绑定服务后返回的接口，在服务端中实现。

具体实现如下：

	private MyBinder mBinder = new MyBinder();

	    private class MyBinder extends Binder
	    {
	        @Override
	        protected boolean onTransact(int code, Parcel data, Parcel reply,
	                int flags) throws RemoteException
	        {
	            switch (code)
	            {
	            case 0x110:
	            {
	                data.enforceInterface(DESCRIPTOR);
	                int _arg0;
	                _arg0 = data.readInt();
	                int _arg1;
	                _arg1 = data.readInt();
	                int _result = _arg0 * _arg1;
	                reply.writeNoException();
	                reply.writeInt(_result);
	                return true;
	            }
	            case 0x111:
	            {
	                data.enforceInterface(DESCRIPTOR);
	                int _arg0;
	                _arg0 = data.readInt();
	                int _arg1;
	                _arg1 = data.readInt();
	                int _result = _arg0 / _arg1;
	                reply.writeNoException();
	                reply.writeInt(_result);
	                return true;
	            }
	            }
	            return super.onTransact(code, data, reply, flags);
	        }

这样就直接使用了Binder机制。通过这些例子，是否更了解Binder一点了呢~感谢大牛的博客和资源！

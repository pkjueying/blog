---
title: binder简要学习
date: 2016-03-17 13:36:49
categories: android
tags: [android,binder]
comment: true
toc: true
description: 理解Binder对于理解整个Android系统有着非常重要的作用，Android系统的四大组件，AMS，PMS等系统服务无一不与Binder挂钩

---


### 本文目标

理解Binder对于理解整个Android系统有着非常重要的作用，Android系统的四大组件，AMS，PMS等系统服务无一不与Binder挂钩；如果对Binder不甚了解，那么就很难了解这些系统机制.

要真正的弄明白 Binder机制 还是比较麻烦的，我们今天只是大致的介绍一下在应用层怎么使用

本文目标:

* 不依赖AIDL工具，手写远程Service完成跨进程通信

* 弄明白AIDL生成的相关代码

* 以及基于AIDL代码的分析，了解系统相关服务的代码逻辑.


### Linux相关概念

因为是讲进程间的通信，而android又是基于linux，所以对于linux系统需要一定的了解.
推荐 [linux内核设计与实现](https://book.douban.com/subject/1503819/) 其主要是讲一些系统概念


* 操作系统的不同进程之间，数据不共享；对于每个进程来说，都以为自己独享了整个系统，完全不知道其他进程的存在；因此一个进程需要与另外一个进程通信，需要某种 **系统机制** 才能完成。

* 用户程序只可以访问某些许可的资源，不许可的资源是拒绝被访问的，于是认为的就把Kernel和上层的应用程序抽像的隔离开，分别称之为 **内核空间(Kernel Space)** 和 **用户空间(User Space)** .

* 用户空间访问内核空间的唯一方式就是 **系统调用** ；通过这个统一入口，所有的资源访问都是在内核的控制下执行，以免导致用户程序对系统资源的越权访问，从而保障了系统的安全和稳定.

* 当一个任务（进程）执行系统调用而陷入内核代码中执行时，我们就称进程处于 **内核运行态** 此时处理器处于特权级最高的内核代码中执行。当进程在执行用户自己的代码时，则称其处于 **用户运行态**（用户态）。处理器在特权等级高的时候才能执行那些特权CPU指令。

* 通过系统调用，用户空间可以访问内核空间. 如果一个用户空间想与另外一个用户空间进行通信，一般是需要操作系统内核添加支持. 

* Linux有个比较好的机制，就是可以 **动态加载内核模块** ；**模块** 是具有独立功能的程序，它可以被单独编译，但不能独立运行。它在运行时被链接到内核作为内核的一部分在内核空间运行。

	这样，Android系统就可以在Linux的基础之上，通过添加一个内核模块运行在内核空间，用户进程之间可通过这个模块完成通信。这个模块就是所谓的 **Binder驱动** .

	尽管名叫‘驱动’，实际上和硬件设备没有任何关系，只是实现方式和设备驱动程序是一样的：它工作于内核态，提供open()，mmap()，poll()，ioctl()等标准文件设备的操作.

* Linux已拥有的进程间通信IPC手段包括： Pipe、Signal、Socket、Message、Share Memory 和信号量Semaphore.

### 为何使用Binder

为什么要单独弄一套， 而不是使用linux系统提供的那些进程间通信的方式

主要是考虑到性能和安全，还有易用. 

* 最易想到的就是利用存储-转发机制：使用Linux 提供的系统调用接口：copy_from_user()和copy_to_user() 来实现进程间通信，缺点是效率低下，需要做两次拷贝：用户空间->内核空间->用户空间。

* Binder驱动 为了实现用户空间到用户空间的拷贝，mmap()分配的内存除了映射进了接收方进程里，还映射进了内核空间。所以调用copy_from_user()将数据拷贝进内核空间也相当于拷贝进了接收方的用户空间， 所以Binder机制只需要一次拷贝。

* 而共享内存的话，效率比较高， 但控制复杂....

* 安全性: 传统IPC没有任何安全措施，完全依赖上层协议来确保；传统IPC访问接入点是开放的，无法建立私有通道；例如Socket通信的话，socket的ip地址或文件名都是开放的，只要知道这些接入点的程序都可以和对端建立连接，不管怎样都无法阻止恶意程序通过猜测接收方地址获得连接。

> Binder基于Client-Server通信模式，传输过程只需一次拷贝，为发送发添加UID/PID身份，既支持实名Binder也支持匿名Binder，安全性高。



### Binder通信模型


应用层大家所熟知的通信结构， 如下图:
    
![Binder通信概述](/img/binder/binder通信概述.jpg)

1. 从表面上来看，是client通过获得一个server的代理接口，对server进行直接调用；
2. 实际上，代理接口中定义的方法与server中定义的方法是一一对应的；
3. client调用某个代理接口中的方法时，代理接口的方法会将client传递的参数打包成为Parcel对象；
4. 代理接口将该Parcel发送给内核中的binder driver.
5. server会读取binder driver中的请求数据，如果是发送给自己的，解包Parcel对象，处理并将结果返回；
6. 整个的调用过程是一个同步过程，在server处理的时候，client会block住。


在整个Binder系统中，Binder框架定义了四个角色：Server，Client，ServiceManager 以及Binder驱动。其中Server，Client，SM运行于用户空间，驱动运行于内核空间

![Binder通信模型](/img/binder/binder-通信模型.jpg)

整个步骤如下: 

* SM建立；首先有一个进程向驱动提出申请为SM；驱动同意之后，SM进程负责管理Service.

* 各个Server向SM注册；将Server自己的名字和内存地址报告给SM; 这样SM就建立了一张表，对应着各个Server的名字和地址

* Client想要与Server通信，首先询问SM；通过服务名获取到对应的内存地址；Client收到之后，就可以进行通信了.

	可以看出驱动是整个通信过程的核心，完成跨进程通信的秘密全部隐藏在驱动里面；这里Client与SM的通信，以及Client与Server的通信，都会经过驱动
	
	
> 相关接口可参见 native/libs/binder/IServiceManager.cpp  以及对应的native 层 service_manager.c


### Binder机制跨进程原理

![Binder机制跨进程原理](/img/binder/binder-跨进程原理.jpg)

* 首先，Server进程要向SM注册；告诉自己是谁，自己有什么能力;在这个场景就是Server告诉SM，它叫AAA，它有一个object对象，可以执行add 操作；于是SM建立了一张表：AAA这个名字对应进程Server; 如原代码中 .//native/libs/binder/IServiceManager.cpp

		virtual status_t addService(const String16& name, const sp<IBinder>& service,
            bool allowIsolated)
	    {
    	    Parcel data, reply;
		data.writeInterfaceToken(IServiceManager::getInterfaceDescriptor());
        	data.writeString16(name);
        	data.writeStrongBinder(service);
	        data.writeInt32(allowIsolated ? 1 : 0);
    	    status_t err = remote()->transact(ADD_SERVICE_TRANSACTION, data, &reply);
        	return err == NO_ERROR ? reply.readExceptionCode() : err;
    	}

* 然后Client向SM查询：名字叫做AAA的进程里面的object对象；进程之间通信的数据都会经过运行在内核空间里面的驱动，驱动在数据流过的时候会做一些处理，它并不会给Client进程返回一个真正的object对象，而是返回一个看起来跟object一模一样的代理对象objectProxy，这个objectProxy也有一个add方法，但是这个add方法没有Server进程里面object对象的add方法那个能力；它唯一做的事情就是把参数包装然后交给驱动。

* 驱动收到这个消息，发现是这个objectProxy；通过查表就知道：之前用objectProxy替换了object发送给Client了，它真正应该要访问的是object对象的add方法；于是Binder驱动通知Server进程，调用它的object对象的add方法，然后把结果发给binder驱动，Sever进程收到这个消息，执行add之后将结果返回驱动，驱动然后把结果返回给Client进程；于是整个过程就完成了.

Binder跨进程传输并不是真的把一个对象传输到了另外一个进程；传输过程是在Binder跨进程穿越的时候，它在一个进程留下了一个本体，在另外一个进程则使用该对象的一个proxy；Client进程的操作其实是对于proxy的操作，proxy利用Binder驱动最终让真正的binder对象完成操作。

Android系统实现这种机制使用的是代理模式, 对于Binder的访问，如果是在同一个进程，那么直接返回原始的Binder实体；如果在不同进程，那么就给他一个代理对象- 在后面的demo中我们可以看见...
	
	public static ICalculate asInterface(IBinder obj) {
            if(obj == null) {
                return null;
            } else {
                IInterface iin = obj.queryLocalInterface("com.zhangfl.jpush.ICalculate");
                return (ICalculate)(iin != null && iin instanceof ICalculate?(ICalculate)iin:new ICalculate.Stub.Proxy(obj));
            }
        }
        

> Client进程只不过是持有了Server端的代理；代理对象协助驱动完成了跨进程通信。

### proxy代理模式

模式中的三种角色:

![proxy设计模式](/img/binder/proxy_uml.jpg)

* 抽象角色：声明真实对象和代理对象的共同接口。

* 代理角色：代理对象角色内部含有对真实对象的引用，从而可以操作真实对象，同时代理对象提供与真实对象相同的接口以便在任何时刻都能代替真实对象。同时，代理对象可以在执行真实对象操作时，附加其他的操作，相当于对真实对象进行封装。

* 真实角色：代理角色所代表的真实对象，是我们最终要引用的对象。

> 模式原则: 对修改关闭，对扩展开放，保证了系统的稳定性



### 驱动里面的Binder


略过:  具体可以参考源码以及 [Binder设计与实现](http://blog.csdn.net/universus/article/details/6211589) 一文


### Java层的Binder

IBinder/IInterface/Binder/BinderProxy/Stub

* IBinder是一个接口，它代表了一种跨进程传输的能力；只要实现了这个接口，就能将这个对象进行跨进程传递；这是驱动底层支持的；在跨进程数据流经驱动的时候，驱动会识别IBinder类型的数据，从而自动完成不同进程Binder本地对象以及Binder代理对象的转换。

* IInterface代表的就是远程server对象具有的能力。具体来说，就是aidl里面的接口。

* Java层的Binder类，代表的其实就是Binder本地对象。BinderProxy类是Binder类的一个内部类，它代表远程进程的Binder对象的本地代理；这两个类都继承自IBinder, 因而都具有跨进程传输的能力；实际上，在跨越进程的时候，Binder驱动会自动完成这两个对象的转换。

* 在使用AIDL的时候，编译工具会给我们生成一个Stub的静态内部类；这个类继承了Binder, 说明它是一个Binder本地对象，它实现了IInterface接口，表明它具有远程Server承诺给Client的能力；Stub是一个抽象类，具体的IInterface的相关实现需要我们手动完成. 其实这里使用了策略模式.

### AIDL过程分析

一种固定的模式：

* 一个需要跨进程传递的对象一定继承自IBinder，如果是Binder本地对象，那么一定继承Binder实现IInterface，如果是代理对象，那么就实现了IInterface并持有IBinder引用；

Proxy与Stub不一样，虽然他们都既是Binder又是IInterface，不同的是Stub采用的是继承（is 关系），Proxy采用的是组合（has 关系）。他们均实现了所有的IInterface函数，不同的是，Stub使用策略模式调用的是虚函数（待子类实现），而Proxy则使用组合模式。为什么Stub采用继承而Proxy采用组合？事实上，Stub本身is一个IBinder（Binder），它本身就是一个能跨越进程边界传输的对象，所以它得继承IBinder实现transact这个函数从而得到跨越进程的能力（这个能力由驱动赋予）。Proxy类使用组合，是因为他不关心自己是什么，它也不需要跨越进程传输，它只需要拥有这个能力即可，要拥有这个能力，只需要保留一个对IBinder的引用


### demo.

......


### 系统服务分析

IXXX、IXXX.Stub和IXXX.Stub.Proxy，并做好对应。这样看相关的系统服务就比较容易了，以ServiceManager为例

实际上ServerManager既是系统服务的管理者，同时也是一个系统服务。因此它肯定是基于Binder实现的

- 与IXXX相对应的类就是IServiceManager类，封装了远程调用的几个主要函数

- 与IXXX.Stub对应的类就是ServiceManagerNative

- 与IXXX.Stub.Proxy对应的类ServiceManagerProxy

	查看上面相关类的代码，实际上和使用adil生成的代码没什么两样。仅仅是类命名不一样，将三个类分开写了而已。

	在服务端继承ServiceManagerNative类实现里面的相关方法就能实现服务端，然后在客户端将远程服务端所对应的的Binder封装成：
	
	IServiceManager iSm = ServiceManagerNative.asInterface(binder)即可

	PS： 实际上，在源码中找不到继承自ServiceManagerNative类的远程服务端类ServiceManagerService，根本就找不到这样一个类。原因是SMS在native层被实现成一个独立的进程，是在启动后解析init.rc脚本启动服务的.
	

再看看ActivityManager中的Binder。

- IActivityManager对应IXXX接口

- ActivityManagerNative对应IXXX.Stub类，继承自Binder类。

- ActivityManagerProxy对应IXXX.Stub.Proxy类。

	AMS的服务端就是ActivityManagerService类，这个类继承自ActivityManagerNative，实现了IActivityManager接口中的方法用来进行IPC。

	只要在客户端得到了这个远程服务端的Binder引用就可以进行IPC通信了



### 参考

* [linux内核设计与实现](https://book.douban.com/subject/1503819/)

* [Binder设计与实现](http://blog.csdn.net/universus/article/details/6211589)

* [Android进程间通信（IPC）机制Binder简要介绍和学习计划系列](http://blog.csdn.net/luoshengyang/article/details/6618363)


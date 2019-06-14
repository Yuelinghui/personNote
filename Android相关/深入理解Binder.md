##深入理解Binder

Binder是Android系统提供的一种IPC（进程间通信）机制。由于Android是基于Linux内核的，因此，除了Binder以外，还有其他的IPC机制，例如socket等。Binder相对于其他IPC机制来说，更加灵活和方便。Android系统基本上可以看做一个基于Binder通信的C/S架构。Binder就像网络一样，把系统的各个部分连接在了一起，因此它是非常重要的。

在基于Binder通信的C/S架构体系中，除了Client端和Server端外，Android还有一个全局的ServiceManager端，它的作用是管理系统中的各种服务（Service）。 它们三者的交互关系如图：

![](http://ofowf99vj.bkt.clouddn.com/Binder%E5%85%B3%E7%B3%BB.jpeg)

* Server进程要先注册一些Service到ServiceManager中，所以Server是ServiceManager的客户端，而ServiceManager就是服务端。
* 如果某个Client进程要使用某个Service，必须先到ServiceManager中获取该Service的相关信息，所以Client是ServiceManager的客户端。
* Client根据得到的Service信息与Service所在的Server进程建立通信的通道，然后就可以直接与Service交互了，所以Client也是Server的客户端。
* 最重要的一点是：**三者的交互都是基于Binder通信的，所以通过任意两者之间的关系，都可以揭示Binder的奥秘。**

**Binder只是为C/S架构提供了一种通信的方式**

####MediaServer

借助MediaServer来解析Binder。MediaServer是系统诸多重要Service的栖息地。
* AudioFinger：音频系统中的核心服务。
* AudioPolicyService：音频系统中关于音频策略的重要服务。
* MediaPlayerService：多媒体系统中的重要服务。
* CameraService：有关摄像/照相的重要服务。

```
// 获取一个ProcessState实例
sp<ProcessState> proc(ProcessState::self());
// MS作为ServiceManager的客户端，需要向ServiceManager注册服务
sp<IserviceManager> sm = defaultServiceManager();
// 初始化AudioFlinger服务
AudioFlinger::instantiate();
// 初始化MediaPlayer服务
MediaPlayerService::instantiate();
// CameraService初始化
CameraService::instantiate();
// AudioPolicy初始化
AudioPolicyService::instantiate();
```

**每个进程只有一个ProcessState对象**

ProcessState构造的时候调用open_driver，作用就是打开/dev/binder这个设备，它是Android在内核中为完成进程间通信而专门设置的一个虚拟设备。

Prcess::self到底干了什么呢？
* 打开/dev/binder设备，这就相当于与内核的Binder驱动有了交互的通道。
* 对返回的fd使用mmap，这样Binder驱动就会分配一块内存来接收数据。
* 由于ProcessState具有唯一性，因此一个进程只打开设备一次。

defaultServiceManager函数的实现在IServiceManager中实现，它会返回一个IServiceManager对象，通过这个对象，我们可以神奇地与另一个进程ServiceManager进行交互。

BpBinder和BBinder都是Android中与Binder通信相关的代表，它们都是从IBinder类中派生出来的。
* BpBinder是客户端用来与Server交互的代理类，p即Proxy的意思。
* BBinder则是与proxy相对的一端，它是proxy交互的目的端。

BpBinder和BBinder是一一对应的，即某个BpBinder只能和对应的BBinder交互。

BpBinder，BBinder这两个类没有任何地方操作ProcessState打开的那个/dev/binder设备，他俩没有和binder设备直接交互。

interface_cast是指针类型转换，它是一个模板函数，返回IServiceManager::asInterface(obj)

IBinder家族的BpBinder和BBinder是与通信业务相关的，那么业务层的逻辑是如何巧妙地架构在Binder机制上的呢？

1. 定义业务逻辑：IServiceManager定义了ServiceManager所提供的服务。
2. 业务与通信挂钩：Android巧妙地通过DECLARE_METE_INTERFACE和IMPLENT宏，将业务和通信牢牢地钩在了一起。

BpServiceManager对象实现了IServiceManager的业务函数，现在又有BpBinder作为通信的代表，接下来的工作就简单了。

调用BpServiceManager的addService，把请求数据打包成data，然后传给BpBinder的transact函数，就相当于把通信的工作交给了BpBinder。

BpBinder中没有任何与Binder设备交互的地方，那是如何参与通信的呢？秘密就在transact函数中。在这里BpBinder把transact工作交给了IPCThreadState。

线程，它是进程中真正干活的伙计，它是劳者，而“劳者一份”就是每个伙计一份

TLS是ThreadLocal Local Storage（线程本地存储空间）的简称，这种空间每个线程都有，而且线程间不共享这些空间。从线程本地存储空间中获得保存在其中的IPCThreadState对象。

每个线程都有一个IPCThreadState，每个IPCThreadState中都有一个mIn（用来接收来自Binder设备的数据），一个mOut（用来存储发往Binder设备的数据）。

IPCThreadState的transact函数实际完成了与Binder通信的工作。

<!--stackedit_data:
eyJoaXN0b3J5IjpbLTExNDIyMTQwNzRdfQ==
-->
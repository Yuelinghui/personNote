##深入理解Binder

Binder是Android系统提供的一种IPC（进程间通信）机制。由于Android是基于Linux内核的，因此，除了Binder以外，还有其他的IPC机制，例如socket等。Binder相对于其他IPC机制来说，更加灵活和方便。Android系统基本上可以看做一个基于Binder通信的C/S架构。Binder就像网络一样，把系统的各个部分连接在了一起，因此它是非常重要的。

在基于Binder通信的C/S架构体系中，除了Client端和Server端外，Android还有一个全局的ServiceManager端，它的作用是管理系统中的各种服务（Service）。 它们三者的交互关系如图：

![](http://ofowf99vj.bkt.clouddn.com/Binder%E5%85%B3%E7%B3%BB.jpeg)

* Server进程要先注册一些Service到ServiceManager中，所以Server是ServiceManager的客户端，而ServiceManager就是服务端。
* 如果某个Client进程要使用某个Service，必须先到ServiceManager中huo
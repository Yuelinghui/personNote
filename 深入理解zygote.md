##深入理解zygote

zygote中文意思是“受精卵”，它和Android系统中的Java世界有着重要关系。而system_server则“人如其名”，系统中重要的Service都驻留于Java世界中。这两个进程分别是Java世界的半边天，任何一个进程的死亡，都会导致Java世界崩溃。

###zygote分析

zygote本事是一个Native的应用程序，与驱动，内核等均无关系。zygote是由init进程根据init.rc文件中的配置项创建的。zygote最初的名字叫“app_process”，这个名字是在Android.mk文件中指定的，但在运行过程中，app_process通过Linux下的pctrl系统调用将自己的名字换成了“zygote”。

原型app_process所对应的源文件是App_main.cpp，mian函数虽然简单，但其重要功能却是由AppRuntime的start来完成的。

###AppRuntime分析

AppRuntime类的声明和实现均在App_main.cpp中，它是从AndroidRuntime类派生出来的。

AppRuntime重载了onStarted，onZygoteInit和onExit函数。

在start中有三个关键点，它们共同组成了开创Android系统中Java世界的三部曲。

```
// 创建虚拟机
startVm(&mJavaVM,&env);
// 注册JNI函数
startReg(env);
/*
  通过JNI调用Java函数，注意调用的函数是main。在调用ZygoteInit的main函数后，zygote变进入了Java世界
*/
env->CallStaticVoidMethod(startClass,startMeth,strArray);
```

* 创建虚拟机-startVm，这个函数没有特别之处，就是调用JNI的虚拟机创建函数，但是创建虚拟机时的一些参数却是在startVm中确定的。
* 注册JNI函数-startReg，因为后续Java世界用到的一些函数式采用native方式实现的，所以才必须提前注册这些函数。
* CallStaticVoidmethod最终将调用com.android.internal.ZygoteInit的main函数，这就是Java世界的入口。

###ZygoteInit的main函数

```
// 注册zygote用的socket
registerZygoteSocket();
...
// 预加载类和资源
preloadClasses();
preloadResources();
...
// 启动system_server进程
startSystemServer();
...
// zygote调用这个函数
runSelectLoopMode();
...
caller.run();
```

ZygoteInit的main函数有5大关键点

* 简历IPC通信服务端-registerZygoteSocket。zygote及系统中其他程序的通信没有使用Binder，而是采用了基于AF_UNIX类型的Socket。这一步就是创建一个服务端的Socket。
* 预加载类和资源。
    * preloadClasses,预加载类的信息存储在PRELOADED_CLASSES变量中，它的值为“preloaded-classes”，通过Java反射来加载类。preload_class文件由framework/base/tools/preload工具生成，它判断每个类加载的时间是否大于1250微秒，超过这个时间的类就会被写到preload-classes文件中，最后由zygote预加载。preloadClass函数的执行时间比较长，这是导致Android系统启动慢的原因之一。
    * preloadResources主要是加载framework-res.apk中的资源。
* 启动system_server。startSystemServer会创建Java世界中系统Service所驻留的进程system_server，该进程是framework的核心，如果它死了，就会导致zygote自杀。zygote进行了一次无性繁殖，分裂出了一个system_server进程（Zygote.forkSystemServer）
* 有求必应之等待请求-runSelectLoopMode。前面注册了一个用于IPC的Socket，它的用途就在这里体现出来。主要是处理客户连接和客户请求。客户在zygote中用ZygoteConnection对象来表示，客户的请求由ZygoteConnection的runOnce来处理。

####关于zygote的总结

zygote是在Android系统中创建Java世界的盘古，它创建了第一个Java虚拟机，同时它又是女娲，它成功地繁殖了framework的核心system_server进程。

* 创建AppRuntime对象，并调用它的start，此后的活动则由AppRuntime来控制。
* 调用startVm创建Java虚拟机，然后调用startReg来注册JNI函数。
* 通过JNI调用com.android.internal.os.ZygoteInit类的main函数，从此进入了Java世界。
* 调用registerZygoteSocket，通过这个函数，它可以响应子孙后代的请求。同时zygote调用preloadClasses和preloadResources。
* zygote觉得自己的工作压力太大，便通过调用startSystemServer分裂一个子进程system_server来为Java世界服务。
* zygote完成了Java世界的初创工作，调用runSelectLoopMode。当接收到子孙后代的请求时，它会随时醒来，为它们工作。

###SystemServer分析

SystemServer的进程名实际上叫做“system_server”，简称为SS。它是由Zygote通过调用函数fork诞生的。

SS诞生后，边和生父Zygote分道扬镳了，它有自己的使命。SS调用handleSystemServerProcess来承担自己的职责。

* zygoteInitNaticve。它是一个native函数，实现在AndroidRuntime.cpp中。SS调用zygoteInitNative后，将与Binder通信系统建立联系，这样SS就能够使用Binder了。
* invokeStaticMain。它抛出一个异常，在ZygoteInit的main函数中被截获，最后会导致com.android.server.SystemServer类的main函数被调用。为什么采用抛异常的方式呢？**这个调用是在ZygoteInit.main中，相当于Native的main函数，即入口函数，位于堆栈的顶层。如果不采用抛异常的方式，而是在invokeStaticMain那里调用，则会浪费之前函数调用所占用的一些调用堆栈。**

####SystemServer真面目

ZygoteInit分裂产生的SS，其实就是为了调用com.android.server.SystemServer的main函数。

main函数将加载libandroid_server.so库。然后调用init1。

init1是native函数，调用system_init。init1创建了一些系统服务，然后把调用线程加入到Binder通信中。还通过JNI调用了init2函数。

init2在Java层。启动了一个ServerThread线程，用于启动系统的各项服务（比如电源管理服务，电池管理服务等等，总之，系统各项重要的服务都在这里启动）。

###zygote的分裂

zygote分裂出system_server后，就通过runSelectLoopMode等待并处理来自客户的消息了。那谁会向zygote发送消息呢？以一个Activity的启动为例，具体分析zygote是如果分裂和繁殖的。

####ActivityManagerService发送请求。

ActivityManagerService也是由SystemServer创建的。通过startActivityManagerService来启动一个新的Activity，而这个ActivityManagerService附属于一个还没启动的进程，那么这个进程如何启动呢？
* ActivityManagerService调用startProcessLocked函数。
* 然后函数里调用ProcessLeica的start（_这里的Process类是Android提供的，并非JDK中的Process类_）
* start方法中调用startViaZygote，处理参数，最后调用zygoteSendArgsAndGetPid。
* 这个方法里连接zygote，把请求参数发送到zygote，然后读取zygote处理完的结果，便可得知是某个进程的pid。

这样ActivityManagerService终于想zygote发送请求了，请求的参数中有一个字符串，它的值是”android.app.ActivityThread“。由于ActivityManagerService驻留于SystemServer进程中，所以正是SS向Zygote发送了消息。

####响应请求

Zygote等待请求，等请求来了，会响应请求，会回到ZygoteInit中。
每当有请求数据发来时，zygote就会调用ZygoteConnection的runOnce函数。

在runOnce函数中，zygote又分裂出了一个子进程，然后处理子进程，根据传入的参数设置新进程的一些属性，最终还是调用invokeStaticMain函数。zygote分裂子进程后，自己做了一些扫尾工作，然后继续等待请求进行下一次分裂。

zygote的分裂由SS控制。

####虚拟机heapsize的限制。

zygote是通过fork来创建子进程的，Zygote本身设置的信息会被子进程全部继承，所以设置Java虚拟机堆栈最大值后，所有的Java程序都会是这个值。

####开机速度优化

Android开机时有三个地方耗时比较长：

1. ZygoteInit的main函数中preloadClasses加载了一千多个类，这耗费了不少时间。
2. 开机启动时，会对系统内所有的apk文件进行扫描并收集信息，这个动作耗费的时间非常长。
3. SystemServer创建的那些Service，会占用不少时间。

使用fork机制来创建Java进程的好处：

1. zygote预加载的这些classes，在fork子进程时，仅需做一个复制即可，这就节约了子进程的启动时间。
2. 根据fork的copy-on-write机制可知，有些累如果不做改变，甚至都不用复制，它们会和父进程共享数据，这样就会省去不少内存的占用。

有人使用Berkeley Lab Checkpoin/Restart(BLCR)技术来提升开机速度，构想就是对系统做一个快照，将系统当前信息保存到一个文件中，当系统重启时，直接根据文件中的快照信息来恢复重启之前的状态。

####Watchdog分析

Android对SystemServer的参数是否被设置也很谨慎，所以专门为它增加了一条看门狗。

Watchdog的调用流程：

1. Wacthdog.getInstance().init();
2. Watchdog.getInstance.start();
3. Watchdog.getInstance.addMonitor();

Watchdog使用单例模式，从线程类中派生，会在一个单独的线程中执行。

init()方法就是初始化。

start()方法导致Watchdog的run在另外一个线程中被执行。**隔一段时间会给另外一个线程发送一条MONITOR消息，那个线程将检查各个Service的健康状况。而看门狗会等待检查结果，如果第二次还没有返回结果，那么它会杀掉SS。**

一个有三个Service是需要交给Watchdog检查的：

1. ActivityManagerService

2. PowerManagerService

3. WindowManagerService

想要支持看门狗的检查，就需要让这些Service实现monitor接口，然后Watchdog就会调用它们的monitor函数进行检查。

monitor检查的就是**这些Service是不是发生死锁了！**

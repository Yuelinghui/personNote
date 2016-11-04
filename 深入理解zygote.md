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
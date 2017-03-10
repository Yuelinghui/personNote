#IntentService原理

##介绍

IntentService主要是用来处理耗时的任务，为什么要使用它：
* 在Service里处理耗时操作容易引起ANR（超过20秒），若要在Service里处理耗时操作，尽量使用IntentService
* IntentService继承自Service，属于四大组件之一。而根据Android系统的特点，系统倾向于释放没有活跃的四大组件的App。将App放入后台，这时候如果内存紧缺，系统就会倾向结束这个App来释放内存。那么这个时候，由于我们有活动的IntentService，优先级较高，就不会被轻易结束。

##原理

先看onCreate()方法：

![](/assets/IntentService_onCreate.jpeg)

首先初始化了一个HandlerThread，然后调用start()方法。

我们来看看HandlerThread类的run()：

![](/assets/HandlerThread_run.jpeg)

替我们创建了一个Looper，方便我们使用Handler。

后面拿到ServiceLooper，然后初始化了一个ServiceHandler。

![](/assets/IntentService_ServiceHandler.jpeg)

也就是说，这个Hanlder是在一个新的线程里面（HanlderThread本质是个Thread）来处理消息的（handleMessage方法）

看看这个handleMessage方法里面有：`onHandleIntent((Intent)msg.obj);`，这个就是我们继承IntentService必须重写的方法，即在这里面进行耗时任务。

然后执行`stopSelf(msg.arg1);`：

![](/assets/IntentService_stopSelf.jpeg)

IntentService在执行完任务之后，就会尽快”自杀”。

IntentService在执行完onCreate()之后就会执行onStartCommand：

![](/assets/IntentService_onStartCommand.jpeg)

调用了onStart。然后就是我们平时使用Hanlder的场景了，通过sendMessage(msg)。然后在Handler中的handleMessage来处理消息了。
刚刚我们也看到了，在handleMessage中又调用了onHandleIntent。
并且这个Handler是运行在一个子线程中的，所以onHandleIntent也是运行在子线程中，这就是为什么IntentService可以用来执行耗时任务的原因了！

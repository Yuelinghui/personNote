# AsyncTask探索

AsyncTask是对Handler与线程池的封装。使用它的方便之处在于能够更新用户界面，当然这里更新用户界面的操作还是在主线程中完成的，但是由于AsyncTask内部包含一个Handler，所以可以发送消息给主线程让它更新UI。另外，AsyncTask内还包含了一个线程池。使用线程池的主要原因是避免不必要的创建及销毁线程的开销。

## AsyncTask局限性

AsyncTask的优点在于执行完后台任务后可以很方便的更新UI，然而使用它存在着诸多的限制。使用AsyncTask主要存在以下局限性：

* 在Android 4.1版本之前，AsyncTask类必须在主线程中加载，这意味着对AsyncTask类的第一次访问必须发生在主线程中；在Android 4.1以及以上版本则不存在这一限制，因为ActivityThread（代表了主线程）的main方法中会自动加载AsyncTask
* AsyncTask对象必须在主线程中创建
* AsyncTask对象的execute方法必须在主线程中调用
* 一个AsyncTask对象只能调用一次execute方法
* 如果AsyncTask被声明为Activity的非静态的内部类，那么AsyncTask会保留一个对创建了AsyncTask的Activity的引用。如果Activity已经被销毁，AsyncTask的后台线程还在执行，它将继续在内存里保留这个引用，导致Activity无法被回收，引起内存泄露。

## AsyncTask工作原理

### AsyncTask构造器

先看AsyncTask的构造器都做了什么：

![](/assets/AsyncTask_Construct.jpeg)

首先，它初始化了mWorker，这是一个WorkerRunnable类的对象。WorkRunnable是一个抽象类，它实现了`Callable<Result>`接口。

我们再来看call方法的定义：
* 首先将mTaskInvoked设为true表示当前任务已被调用过
* 然后设置线程的优先级
* 然后，调用了AsyncTask对象的doInBackground方法开始执行我们所定义的后台任务，并获取返回结果存入result中
* 最后将任务返回结果传递给postResult方法。

由此我们可以知道，**实际上AsyncTask的成员mWorker包含了AyncTask最终要执行的任务（即mWorker的call方法）**。

接下来让我们看看对mFuture的初始化。我们可以看到mFuture是一个FutureTask的直接子类（匿名内部类）的对象，在FutureTask的构造方法中我们传入了mWorker作为参数。

![](/assets/FutureTask_Constructor.jpeg)

mFuture是一个封装了我们的后台任务的FutureTask对象，FutureTask类实现了FutureRunnable接口，通过这个接口可以方便的取消后台任务以及获取后台任务的执行结果。

### AsyncTask执行任务

当mWorker中定义的call方法被执行时，doInBackground就会开始执行，我们定义的后台任务也就真正开始了。那么这个call方法什么时候会被调用呢？我们可以看到经过层层封装，实际上是mFuture对象封装了call方法，当mFuture对象被提交到AsyncTask包含的线程池执行时，call方法就会被调用，我们定义的后台任务也就开始执行了。

我们来看看execute方法：

![](/assets/AsyncTask_execute.jpg)

它接收的参数是Params类型的参数，这个参数会一路传递到doInBackground方法中。execute方法仅仅是调用了executeOnExecutor方法，并将executeOnExecutor方法的返回值作为自己的返回值。

我们注意到，传入了sDefaultExecutor作为executeOnExecutor方法的参数，那么sDefaultExecutor是什么呢？简单的说，它是AsyncTask的默认执行器（线程池）

我们再来看executeOnExecutor（）方法：

![](/assets/AsyncTask_executeOnExecutor.jpeg)

当AsyncTask对象的当前状态为RUNNING或FINISHED时，调用execute方法会抛出异常，这意味着不能对正在执行任务的AsyncTask对象或是已经执行完任务的AsyncTask对象调用execute方法，这也就解释了我们上面提到的局限中的最后一条。

存在一个对onPreExecute方法的调用，这表示了在执行后台任务前确实会调用onPreExecute方法。

之后，把我们传入的execute方法的params参数赋值给了mWorker的mParams成员变量；而后调用了exec的execute方法，并传入了mFuture作为参数。exec就是我们传进来的sDefaultExecutor。

```
private static volatile Executor sDefaultExecutor = SERIAL_EXECUTOR;

public static final Executor SERIAL_EXECUTOR = new SerialExecutor();
```

实际上sDefaultExecutor是一个SerialExecutor对象。

![](/assets/AsyncTask_SerialExecutor.jpg)

mTasks代表了SerialExecutor这个串行线程池的任务缓存队列，我们用offer方法向任务缓存队列中添加一个任务，任务的内容如run方法定义所示。

我们可以看到，run方法中：调用了mFuture（r就是我们传入的mFuture）的run方法，而mFuture的run方法内部会调用mWorker的call方法，然后就会调用doInBackground方法，我们的后台任务也就开始执行了。那么我们提交到任务缓存队列中的任务什么时候会被执行呢？我们接着往下看。

首先我们看到定义了一个Runnable变量mActive，它代表了当前正在执行的AsyncTask对象。判断mActive是否为null，若为null，就调用scheduleNext方法。在scheduleNext方法中，若缓存队列非空，则调用THREAD_POOL_EXECUTOR.execute方法执行从缓存队列中取出的任务，这时我们的后台任务便开始你真正执行了。

通过以上的分析，我们可以知道SerialExecutor所完成的工作主要是把任务加到任务缓存队列中，而真正执行任务的是**THREAD_POOL_EXECUTOR。**

![](/assets/thread_pool_executor.jpeg)

它是一个线程池对象。根据AsyncTask的源码，我们可以获取它的各项参数：

![](/assets/thread_pool_executor_参数.jpeg)

由以上代码我们可以知道：
* corePoolSize大小
* maximumPoolSize为CPU数的二倍加一
* 存活时间为30秒
* 任务缓存队列为LinkedBlockingQueue。

### AsyncTask获取结果

我们已经了解到了从我们调用AsyncTask对象的execute方法开始知道后台任务执行完都发生了什么。现在让我们来看一看postResult方法的源码：

![](/assets/postResult.jpeg)

先调用了getHandler方法获取AsyncTask对象内部包含的sHandler，然后通过它发送了一个MESSAGE_POST_RESULT消息

![](/assets/getHandler.jpeg)

![](/assets/internalHandler.jpeg)

sHandler是一个静态的Handler对象。我们知道创建Handler对象时需要当前线程的Looper，所以我们为了以后能够通过sHandler将执行环境从后台线程切换到主线程（即在主线程中执行handleMessage方法），我们必须使用主线程的Looper，因此必须在主线程中创建sHandler。这也就解释了为什么必须在主线程中加载AsyncTask类，是为了完成sHandler这个静态成员的初始化工作。

当sHandler收到MESSAGE_POST_RESULT方法后，会调用finish方法：

![](/assets/AsyncTask_finish.jpeg)

首先会通过调用isCancelled方法判断AsyncTask任务是否被取消，若取消了则调用onCancelled方法，否则调用onPostExecute方法；最后，把mStatus设为FINISHED，表示当前AsyncTask对象已经执行完毕。

<!--stackedit_data:
eyJoaXN0b3J5IjpbLTMxNTM5ODg3M119
-->
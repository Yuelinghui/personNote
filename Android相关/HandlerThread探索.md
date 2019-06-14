# HandlerThread探索

HandlerThread是什么东西呢？了解一个类最好的方法就是查看类的定义，所以我们就看一下HandlerThread是如何定义的吧。查看类的定义时有这样一段话：

![](/assets/HandlerThread_annote.jpeg)

**这个类的作用是创建一个包含looper的线程。**

假如在应用程序当中为了实现同时完成多个任务，所以我们会在应用程序当中创建多个线程。为了让多个线程之间能够方便的通信，我们会使用Handler实现线程间的通信。这个时候我们手动实现的多线程+Handler的简化版就是我们HandlerThrea所要做的事了。

## Handler基本使用

```
/**
 * 测试HandlerThread的基本使用
 */
    HandlerThread mHandlerThread = new HandlerThread("myHandlerThreand");
    mHandlerThread.start();

    // 创建的Handler将会在mHandlerThread线程中执行
    final Handler mHandler = new Handler(mHandlerThread.getLooper()) {
            @Override
            public void handleMessage(Message msg) {
                Log.i("tag", "接收到消息：" + msg.obj.toString());
            }
    };

    title = (TextView) findViewById(R.id.title);
    title.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                Message msg = new Message();
                msg.obj = "11111";
                mHandler.sendMessage(msg);

                msg = new Message();
                msg.obj = "2222";
                mHandler.sendMessage(msg);
           }
   });
```

首先定义了一个HandlerThread对象，是直接通过new的方式产生的。

HandlerThread的构造函数是：

```
public HandlerThread(String name) {
        super(name);
        mPriority = Process.THREAD_PRIORITY_DEFAULT;
}
```

HandlerThread继承于Thread，所以说HandlerThread本质上是一个线程，其构造方法主要是做一些初始化的操作。

然后我们调用了mHandlerThread.start\(\)方法，由上我们知道了HandlerThread类其实就是一个Thread，一个线程，所以其start方法内部调用的肯定是Thread的run方法：

```
    @Override
    public void run() {
        mTid = Process.myTid();
        Looper.prepare();
        synchronized (this) {
            mLooper = Looper.myLooper();
            notifyAll();
        }
        Process.setThreadPriority(mPriority);
        onLooperPrepared();
        Looper.loop();
        mTid = -1;
    }
```

在我们创建的HandlerThread线程中我们创建了该线程的Looper与MessageQueue

在调用Looper.loop()方法之前调用了一个空的实现方法：onLooperPrepared(),我们可以实现自己的onLooperPrepared（）方法，做一些Looper的初始化操作

run方法里面当mLooper创建完成后有个notifyAll()，getLooper()中有个wait()，这是为什么呢？因为mLooper在一个线程中执行，而handler是在UI线程初始化的，也就是说，我们必须等到mLooper创建完成，才能正确的返回getLooper()；wait(),notify()就是为了解决这两个线程的同步问题

然后我们创建了一个新的Handler，该Handler的构造方法中传入了HandlerThread的Looper对象，所以Handler对象就相当于含有了HandlerThread线程中Looper对象的引用。

然后我们调用handler的sendMessage方法发送消息，在Handler的handleMessge方法中就可以接收到消息了。

最后需要注意的是在我们不需要这个looper线程的时候需要手动停止掉：

```
protected void onDestroy() {
        super.onDestroy();
        mHandlerThread.quit();
}
```

## 总结

* HandlerThread本质上就是一个Thread线程，只不过里面包含了Looper和MessageQueue
* 通过HandlerThread我们不但可以实现UI线程与子线程的通信同样也可以实现子线程与子线程之间的通信
* HandlerThread在不需要使用的时候需要手动的回收掉

<!--stackedit_data:
eyJoaXN0b3J5IjpbLTkwNTkxMTAxMl19
-->
#Android子线程真的不能更新UI吗

Android单线程模型是这样描述的：

**Android UI操作并不是线程安全的，并且这些操作必须在UI线程执行**

为什么呢？在子线程中就不能操作UI么？

当一个程序第一次启动的时候，Android会同时启动一个**对应的主线程，这个主线程就是UI线程**，也就是ActivityThread。UI线程主要负责处理与UI相关的事件，如用户的按键点击、用户触摸屏幕以及屏幕绘图等。系统不会为每个组件单独创建一个线程，在同一个进程里的UI组件都会在UI线程里实例化，系统对每一个组件的调用都从UI线程分发出去。所以，响应系统回调的方法永远都是在UI线程里运行，如响应用户动作的onKeyDown()的回调。

那为什么选择一个主线程干这些活呢？换个说法，Android为什么使用单线程模型，它有什么好处？

**采用一个专门的线程从队列中抽取事件，并把他们转发给应用程序定义的事件处理器**。这就是Android的消息队列、Looper和Handler。

其实现代GUI框架就是使用了类似这样的模型：模型创建一个专门的线程，事件派发线程来处理GUI事件。单线程化也不单单存在Android中，Qt、XWindows等都是单线程化。当然，也有人试图用多线程的GUI，最终由于竞争条件和死锁导致的稳定性问题等，又回到单线程化的事件队列模型老路上来。单线程化的GUI框架通过限制来达到线程安全：所有GUI中的对象，包括可视组件和数据模型，都只能被事件线程访问。

那Android的UI操作并不是线程安全的又是怎么回事？

Android实现View更新有两组方法，分别是invalidate和postInvalidate。前者在UI线程中使用，后者在非UI线程中使用。换句话说，Android的UI操作不是线程安全可以表述为invalidate在子线程中调用会导致线程不安全。作一个假设，现在我用invalidate在子线程中刷新界面，同时UI线程也在用invalidate刷新界面，这样会不会导致界面的刷新不能同步？既然刷新不同步，那么invalidate就不能在子线程中使用。这就是invalidate不能在子线程中使用的原因。

如果非要在子线程中更新UI，那会出现什么情况呢？

```
android.view.ViewRoot$CalledFromWrongThreadException: Only the original thread that created a view hierarchy can touch its views.
```
抛了一个CalledFromWrongThreadException异常。

为什么会出现这个异常呢？这个异常在哪里抛出来的呢？

![](/assets/子线程_CheckThread.jpeg)

这段代码是ViewRootImpl类里的，这里的mThread是从哪初始化的呢？

![](/assets/子线程_ViewRootImpl.jpeg)

而这个checkThread()方法，在很多地方都会调用，比如：`requestFitSystemWindows`，`requestLayout`，`invalidateChildInParent`，这几个方法里都会先调用checkThread方法。

说到底，非UI线程是可以刷新UI的呀，前提是它要拥有自己的ViewRoot。如果想直接创建ViewRoot实例，你会发现找不到这个类。那怎么做呢？通过WindowManager。

```
class NonUiThread extends Thread{
      @Override
      public void run() {
         Looper.prepare();
         TextView tx = new TextView(MainActivity.this);
         tx.setText("non-UiThread update textview");
 
         WindowManager windowManager = MainActivity.this.getWindowManager();
         WindowManager.LayoutParams params = new WindowManager.LayoutParams(
             200, 200, 200, 200, WindowManager.LayoutParams.FIRST_SUB_WINDOW,
                 WindowManager.LayoutParams.TYPE_TOAST,PixelFormat.OPAQUE);
         windowManager.addView(tx, params); 
         Looper.loop();
     }
 }
```

通过windowManager.addView创建了ViewRoot。

Activity.getWindowManager()返回的实例是WindowManagerImpl，然后调用了它的addView方法：

![](/assets/子线程_WindowManagerImpl_addView.jpeg)

调用了mGlobal的addView方法，mGlobal是WindowManagerGlobal的实例。

![](/assets/WindowManagerGlobal_addView.jpeg)

所以，非UI线程能更新UI，只要它有自己的ViewRoot。

Android Activity本身是在什么时候创建ViewRoot的呢？

就在handleResumeActivity里

![](/assets/ActivityThread_handleResumeActivity.jpeg)

这就是Android为我们设计的单线程模型，核心就是一句话：Android UI操作并不是线程安全的，并且这些操作必须在UI线程执行。但这一句话背后，却隐藏着我们平时看不见的代码实现，只有搞懂这些，我们才能知其然知其所以然。


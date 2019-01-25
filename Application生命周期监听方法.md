# Application生命周期监听方法

我们在开发的时候，有时会遇到一种这样的需求：**在应用从后台进入到前台的时候，需要做一些操作**。

遇到这种需求，我们第一想法是：在BaseActivity的onResume里添加这些操作，但是这样写，会造成很大的浪费，因为用户在前台的时候，是不需要这种操作的。要是这么写，在每个activity`onResume`的时候都要执行这些操作。

那有没有什么方法可以监听到Application的生命周期呢？有！使用`ActivityLifecycleCallbacks`和`ComponentCallbacks2`来完成这个任务

## ActivityLifecycleCallbacks

```
public interface ActivityLifecycleCallbacks {
        void onActivityCreated(Activity activity, Bundle savedInstanceState);
        void onActivityStarted(Activity activity);
        void onActivityResumed(Activity activity);
        void onActivityPaused(Activity activity);
        void onActivityStopped(Activity activity);
        void onActivitySaveInstanceState(Activity activity, Bundle outState);
        void onActivityDestroyed(Activity activity);
    }
```
这个接口会在Activity运行生命周期的时候运行，并把当前发生的Activity对象传进来

## ComponentCallbacks2

这是一个细粒度的内存回收管理回调。Application、Activity、Service、ContentProvider、Fragment实现了ComponentCallback2接口，开发者实现onTrimMemory(int)方法，可以细粒度release 内存，参数可以体现不同程度的内存可用情况.

响应onTrimMemory回调，开发者的app会直接受益，有利于用户体验，系统更有可能让app存活的更持久。不响应onTrimMemory回调，系统更有可能kill 进程

```
    override fun onLowMemory() {
    }

    override fun onConfigurationChanged(newConfig: Configuration?) {
    }

    override fun onTrimMemory(level: Int) {
    }
```

就是第三个方法，回调的时候会返回level，这个level对应的值是：

当应用进程被置于background LRU list中时：

* TRIM_MEMORY_COMPLETE // 系统运行慢，当前进程是第一批将被系统kill的进程。此app应该释放一切可以释放的资源。低于api 14的，用户可以使用onLowMemory回调

* TRIM_MEMORY_MODERATE // 系统运行缓慢，当前进程已经位于LRU list的中部，如果系统进一步变慢，便会有被kill的可能

* TRIM_MEMORY_BACKGROUND // 系统运行慢，并且进程位于LRU list的上端。尽管app不处于高风险被kill。当前app应该释放那些容易恢复的资源

当应用的visibility改变时：

* TRIM_MEMORY_UI_HIDDEN // 当前app UI不再可见，这是一个回收大个资源的好时机

当应用正在运行时：

* TRIM_MEMORY_RUNNING_CRITICAL // 设备运行特别慢，当前app还不会被杀死，但是如果此app没有释放资源，系统将会kill后台进程

* TRIM_MEMORY_RUNNING_LOW // 设备运行更缓慢了，当前app正在运行，不会被kill。但是请回收unused资源，以便提升系统的性能

* TRIM_MEMORY_RUNNING_MODERATE // 设备开始运行缓慢，当前app正在运行，不会被kill

现在这个监听方法的level会返回什么值已经知道了，那我们怎么监听呢？

## 监听方案

```
class LiveAppLefecycleHandler : Application.ActivityLifecycleCallbacks,ComponentCallbacks2 {
    private var isInbackground = false

    ...
    override fun onActivityResumed(activity: Activity?) {
        if (isInbackground) {
            // 从后台返回到前台
            isInbackground = false
        }
    }
    ...

     override fun onTrimMemory(level: Int) {
        if (level >= ComponentCallbacks2.TRIM_MEMORY_UI_HIDDEN) {
            // 从前台推到后台
            isInbackground = true
        }
    }
}
```

在`onTrimMemory`中判断level的时候用的是`>=`，是因为有的手机从前台推到后台的时候会返回不同的level，但是肯定都是`TRIM_MEMORY_UI_HIDDEN`以上的数值，所以用`>=`判断

写完这个Handler之后，我们还需要注册它：

```
public class LiveApplication extends MultiDexApplication {
    @Override
    public void onCreate() {
        super.onCreate();
        // 注册Application生命周期监听的Handler
        LiveAppLefecycleHandler handler = new LiveAppLefecycleHandler();
        registerActivityLifecycleCallbacks(handler);
        registerComponentCallbacks(handler);
    }
}
```

## 总结

以后遇到类似的需求，就可以使用这个方案了。同时，我们也可以扩展一下，既然有这个callback，那我们以后在开发的过程中，也可以用这个来释放内存，保证应用在后台可以存活的更长，可能应用后台保活就使用的这个方案吧。

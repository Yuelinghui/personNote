# Lifecycle的使用

生命周期感知组件执行操作以相应另一个组件（例如Activity或Fragment）的生命周期状态的改变。这些组件可以帮助我们生成更易于组织且通常更轻量级的代码，这些代码更易于维护

比较常见的模式：在Activity或Fragment的生命周期方法中实现依赖组件的操作。但是这种模式导致代码组织不良以及错误的增加。通过使用生命周期感知组件，我们可以将依赖组件的代码移出生命周期方法并移入组件本身。

Android框架定义的大多数应用程序组件都附加了生命周期，它由操作系统或流程中运行的框架代码管理。比如我们显示位置：

```
internal class MyLocationListener(
        private val context: Context,
        private val callback: (Location) -> Unit
) {

    fun start() {
        // connect to system location service
    }

    fun stop() {
        // disconnect from system location service
    }
}

class MyActivity : AppCompatActivity() {
    private lateinit var myLocationListener: MyLocationListener

    override fun onCreate(...) {
        myLocationListener = MyLocationListener(this) { location ->
            // update UI
        }
    }

    public override fun onStart() {
        super.onStart()
        myLocationListener.start()
        // manage other components that need to respond
        // to the activity lifecycle
    }

    public override fun onStop() {
        super.onStop()
        myLocationListener.stop()
        // manage other components that need to respond
        // to the activity lifecycle
    }
}
```

即使这个示例看起来很好，但在实际开发中，我们可能会有太多的调用来管理UI和其他组件以响应生命周期的状态变化。一旦管理的组件多起来，会在生命周期方法中放置大量代码，这就使得代码难以维护了。

另外，无法保证组件在Activity或Fragment停止之前启动。若我们需要执行长时间运行的操作，尤其是在`onStart()`中，这可能导致竞争条件，当`onStop()`执行的时候`onStart()`里的操作还没有执行，就会使组件的生命周期长于本来的时间，甚至导致内存泄漏：

```
class MyActivity : AppCompatActivity() {
    private lateinit var myLocationListener: MyLocationListener

    override fun onCreate(...) {
        myLocationListener = MyLocationListener(this) { location ->
            // update UI
        }
    }

    public override fun onStart() {
        super.onStart()
        Util.checkUserStatus { result ->
            // what if this callback is invoked AFTER activity is stopped?
            if (result) {
                myLocationListener.start()
            }
        }
    }

    public override fun onStop() {
        super.onStop()
        myLocationListener.stop()
    }

}
```

上面的例子，`checkUserStatus`是一个耗时操作，当`onStop()`执行的时候，回调还没有返回，当回调返回的时候去执行`listener.onStart()`，就导致组件的生命周期错乱了。

Lifecycle提供了类和接口，可以帮助我们以弹性和隔离的方式解决这些问题

## Lifecycle

Lifecycle是一个类，包含有关组件生命周期状态的信息，并允许其他对象观察此状态。

Lifecycle使用两个主要枚举类跟踪其关联组件的生命周期状态：

### Event

从框架和Lifecycle类调度的生命周期事件。这些事件映射到Activity和Fragment的回调事件中

### State

Lifecycle对象跟踪的组件的当前状态

一个类可以通过像其方法添加注解来监视组件的生命周期状态。然后，可以通过调用类的`addObserver()`方法来传递观察者实例来添加观察者。

```
class MyObserver : LifecycleObserver {

    @OnLifecycleEvent(Lifecycle.Event.ON_RESUME)
    fun connectListener() {
        ...
    }

    @OnLifecycleEvent(Lifecycle.Event.ON_PAUSE)
    fun disconnectListener() {
        ...
    }
}
```

## LifecycleOwner

LifecycleOwner是一个单一的方法接口，表示该类有一个Lifecycle。

任何自定义类都可以实现LifecycleOwner接口。

```
class MyActivity : AppCompatActivity() {
    private lateinit var myLocationListener: MyLocationListener

    override fun onCreate(...) {
        myLocationListener = MyLocationListener(this, lifecycle) { location ->
            // update UI
        }
        Util.checkUserStatus { result ->
            if (result) {
                myLocationListener.enable()
            }
        }
    }
}

internal class MyLocationListener(
        private val context: Context,
        private val lifecycle: Lifecycle,
        private val callback: (Location) -> Unit
) {

    private var enabled = false

    @OnLifecycleEvent(Lifecycle.Event.ON_START)
    fun start() {
        if (enabled) {
            // connect
        }
    }

    fun enable() {
        enabled = true
        if (lifecycle.currentState.isAtLeast(Lifecycle.State.STARTED)) {
            // connect if not connected
        }
    }

    @OnLifecycleEvent(Lifecycle.Event.ON_STOP)
    fun stop() {
        // disconnect if connected
    }
}
```

通过这种实现，我们的listener完全可以识别生命周期。

### 实现自定义LifecycleOwner

```
class MyActivity : AppCompatActivity(), LifecycleOwner {

    private lateinit var mLifecycleRegistry: LifecycleRegistry

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)

        mLifecycleRegistry = LifecycleRegistry(this)
        mLifecycleRegistry.markState(Lifecycle.State.CREATED)
    }

    public override fun onStart() {
        super.onStart()
        mLifecycleRegistry.markState(Lifecycle.State.STARTED)
    }

    override fun getLifecycle(): Lifecycle {
        return mLifecycleRegistry
    }
}
```


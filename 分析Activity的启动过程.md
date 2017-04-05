#分析Activity的启动过程

通过源码分析Activity的启动过程

先从startActivity开始

![](/assets/Activity_startActivity_1.jpeg)
![](/assets/Activity_startActivity_2.jpeg)

所以，所有的启动Activity其实最后走的都是startActivityForResult，只不过startActivity传入的requestCode是-1

之后我们看startActivityForResult方法：

![](/assets/Activity_startActivityForResult.jpeg)

这里是通过Instrumentation的execStartActivity方法来启动Activity：

![](/assets/Instrumentation_execStartActivity.jpeg)

whoThread是一个实现了IApplicationThread接口的类的实例，IApplicationThread接口提供了系统跟应用(application)交互的API

Intent的migrateExtraStreamToClipData()方法应该是判断是否启动系统Activity的，比如发送邮件，拍照等

Intent的prepareToLeaveProcess()方法是为了在离开一个app的process之前做准备

之后调用IActivityManager接口的startActivity方法，把之前的IApplicationThread对象whoThread传入了这个方法。Activity Manager通过该实例指导应用进行诸如启动、暂停、停止Activity等一系列操作

启动Activity的方法是IApplicationThread内的scheduleLaunchActivity，具体的实现是在ActivityThread的私有类ApplicationThread里：

![](/assets/ActivityThread_scheduleLaunchActivity.jpeg)

首先更新一下进程的状态，然后封装了一个ActivityClientRecord对象，包括传入Activity的intent，标识该Activity的token，存储AndroidManifest.xml中`<activity>`和`<receiver>`标记段信息的activityInfo等。之后更新了存储系统级别和用户级别的配置信息curConfig（如屏幕方向，当前国家和字体缩放设置等，通过getResources().getConfiguration()获得）。最后通过sendMessage发送一个类型为LAUNCH_ACTIVITY的message。

这个message是在H里获得的，H是一个继承了Handler的类，里面通过handleMessage来处理这个消息：

![](/assets/H_handleMessage.jpeg)

先从Message里拿到之前封装的ActivityClientRecord对象，然后获取packageInfo，最后调用handleLaunchActivity：

![](/assets/handleLaunchActivity.jpeg)

主要包括三个操作，performLaunchActivity启动Activity，handleResumeActivity来Resume Activity，最后通过Instrumentation的callActivityOnPause函数来OnPause Activity。

performLaunchActivity里首先通过ClassLoader来调用Instrumentation的newActivity方法：

```
Activity activity = null;
try {
            java.lang.ClassLoader cl = r.packageInfo.getClassLoader();
            activity = mInstrumentation.newActivity(cl, component.getClassName(), r.intent);
            StrictMode.incrementExpectedActivityCount(activity.getClass());
            r.intent.setExtrasClassLoader(cl);
            r.intent.prepareToEnterProcess();
            if (r.state != null) {
                r.state.setClassLoader(cl);
            }
    } catch (Exception e) {
            if (!mInstrumentation.onException(activity, e)) {
                throw new RuntimeException(
                    "Unable to instantiate activity " + component
                    + ": " + e.toString(), e);
            }
    }
```

```
public Activity newActivity(ClassLoader cl, String className,
    Intent intent) throws InstantiationException, IllegalAccessException,
            ClassNotFoundException {
        return (Activity)cl.loadClass(className).newInstance();
    }
```

然后通过createBaseContextForActivity生成Activity的context，并通过activity.attach方法将Activity的context和其他属性赋给Activity。

```
Context appContext = createBaseContextForActivity(r, activity);
CharSequence title = r.activityInfo.loadLabel(appContext.getPackageManager());
Configuration config = new Configuration(mCompatConfiguration);
if (r.overrideConfig != null) {
        config.updateFrom(r.overrideConfig);
}
               
Window window = null;
if (r.mPendingRemoveWindow != null && r.mPreserveWindow) {
        window = r.mPendingRemoveWindow;
        r.mPendingRemoveWindow = null;
        r.mPendingRemoveWindowManager = null;
}
activity.attach(appContext, this, getInstrumentation(), r.token,
        r.ident, app, r.intent, r.activityInfo, title, r.parent,
        r.embeddedID, r.lastNonConfigurationInstances, config,
        r.referrer, r.voiceInteractor, window);
```

接着调用Instrumentation的 callActivityOnCreate方法， callActivityOnRestoreInstanceState 方法和callActivityOnPostCreate 方法， 分别对应Activity的onCreate，onRestoreInstanceState和onPostCreate方法。

![](/assets/performLaunchActivity_Instrumentation.jpeg)

这样就在performLaunchActivity函数中完成了Activity的加载和初始化过程。

在performLaunchActivity完成后，通过handleResumeActivity方法使得Activity完成onResume的准备工作，进入onResume状态。最后，调用Instrumentation的callActivityOnPause方法，使得Activity进入onPause的状态，这样，当Activity最终展示出来的时候，就会从onResume作为开始。至此，Activity加载完毕。

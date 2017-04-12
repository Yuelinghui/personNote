# Android targetSdkVersion原理

compileSdkVersion和minSdkVersion都非常好理解，前者表示编译的SDK版本，后者表示应用兼容的最低 SDK 版本。但是对于targetSdkVersion其实很难一句话解析清楚。

targetSdkVersion是Android系统提供前向兼容的主要手段。这是什么意思呢？随着Android系统的升级，某个系统的API或者模块的行为可能会发生改变，但是为了保证老APK的行为还是和以前兼容。只要APK的targetSdkVersion不变，即使这个APK安装在新 Android系统上，其行为还是保持老的系统上的行为，这样就保证了系统对老应用的前向兼容性。

## 举例

在Android 4.4(API 19以后，AlarmManager的`set()`和`setRepeat()`这两个API的行为发生了变化。在Android 4.4以前，这两个API设置的都是精确的时间，系统能保证在API设置的时间点上唤醒Alarm。因为省电原因Android 4.4系统实现了AlarmManager的对齐唤醒，这两个API设置唤醒的时间，系统都对待成不精确的时间，系统只能保证在你设置的时间点之后某个时间唤醒。

这时，虽然API没有任何变化，但是实际上API的行为却发生了变化，如果老的APK中使用了此API，并且在应用中的行为非常依赖AlarmManager在精确的时间唤醒，例如闹钟应用。如果Android系统不能保证兼容，老的APK安装在新的系统上，就会出现问题。

Android系统是怎么保证这种兼容性的呢？这时候targetSdkVersion就起作用了。APK在调用系统AlarmManager的`set()`或者`setRepeat()`的时候，系统首先会查一下调用的APK的 targetSdkVersion 信息，如果小于19，就还是按照老的行为，即精确设置唤醒时间，否者执行新的行为。

![](/assets/AlarmManager_constructor.jpeg)

看到这里，首选获取应用的targetSdkVersion，判断是否是小于Build.VERSION_CODES.KITKAT (即 API Level 19)，来设置`mAlwaysExact`变量，表示是否使用精确时间模式。

![](/assets/AlarmManager_legacyExactLength.jpeg)

![](/assets/AlarmManager_set.jpeg)

这里看到，直接影响到 set() 方法给 setImpl() 传入不同的参数，从而影响到了 set() 的执行行为。

看到这里，发现其实Android的targetSdkVersion并没有什么特别的，系统使用它也非常直接，甚至很“粗糙”。仅仅是用过下面的API来获取targetSdkVersion，来判断是否执行哪种行为：

```
mTargetSdkVersion = ctx.getApplicationInfo().targetSdkVersion;
```

所以，我们可以猜测到，如果Android系统升级，发生这种兼容行为的变化时，一般都会在原来的保存新旧两种逻辑，并通过`if-else`方法来判断执行哪种逻辑。果然，在源码中搜索，我们会发现不少类似`getApplicationInfo().targetSdkVersion < Buid.XXXX`这样的代码，相对于浩瀚的Android源码量来说，这些还是相对较少了。其实原则上，这种会导致兼容性问题的修改还是越少越好，所以每次发布新的Android版本的时候，Android开发者网站都会列出做了哪些改变，在这里，开发者需要特别注意。


##Android N 适配

###权限更改

Android6.0引入了动态权限控制，7.0使用了”私有目录被限制访问“，”Strict Mode API 政策”，这些更改在为用户提供一个更加安全的操作系统的同时，也给开发者带来了新的任务。

####目录被限制访问

在Android中应用可以读写手机存储中任何一个目录和文件，这给系统安全带来了很多问题。

7.0中为了提高私有文件的安全性，面向7.0及更高版本的应用私有目录将被限制访问。

* 私有文件的文件权限不在放权给所有的应用，在manifest里使用**MODE_WORLD_READABLE**或**MODE_WORL_WRITEABLE**进行的操作将触发**SecurityException。**

_应对策略：这项权限的更改将意味着你无法通过File API来访问手机存储上的数据了，基于File API的一些文件浏览等也将受到很大的限制。不过迄今为止，这种限制尚不能完全执行，应用仍可能使用File API来修改它们的私有文件权限。但是，Android官方强烈反对放宽私有目录的权限。_

* 给其他应用传递file://URI这种URI类型，可能导致接收者无法访问该路径。因此，在7.0中尝试传递file://URI会触发**FileUriExposedException。**

_应对策略：可以使用FileProvider来解决这个问题。_

* DownloadManager不再按文件名分享死人存储的路径。**COLUMN_LOCAL_FILENAME**在7.0中被标记为_deprecated_，7.0及更高版本的应用在尝试访问时会触发**SecurityException。**

_应对策略：可以通过ContentProvider.openFileDescriptor()来访问由DownloadManager公开的文件。_

###应用间共享文件

在Android7.0版本上，Android系统强制执行了**StrictMode API 政策**，禁止向你的应用外公开File://URI。如果一项包含文件File://URI类型的Intent离开你的应用，应用失败，并出现**FileUriExposedException**，比如**系统相机拍照，裁剪照片**

####在Android 7.0系统调用相机拍照，裁剪照片

在7.0之前调用系统相机拍照：

```
File file=new File(Environment.getExternalStorageDirectory(),
"/temp/"+System.currentTimeMillis() + ".jpg"); 
if (!file.getParentFile().exists()) {
    file.getParentFile().mkdirs();
}
Uri imageUri = Uri.fromFile(file); 
Intent intent = new Intent(); intent.setAction(MediaStore.ACTION_IMAGE_CAPTURE);//设置Action为拍照 intent.putExtra(MediaStore.EXTRA_OUTPUT, imageUri);//将拍取的照片保存到指定URI startActivityForResult(intent,1006);
```

在7.0上会抛出异常：

```
android.os.FileUriExposedException:file:////storage/emulated/0/temp/1474956193735.jpg exposed beyond app through Intent.getData() 
at android.os.StrictMode.onFileUriExposed(StrictMode.java:1799) 
at android.net.Uri.checkFileUriExposed(Uri.java:2346) 
at android.content.Intent.prepareToLeaveProcess(Intent.java:8933) 
at android.content.Intent.prepareToLeaveProcess(Intent.java:8894) 
at android.app.Instrumentation.execStartActivity(Instrumentation.java:1517) 
at android.app.Activity.startActivityForResult(Activity.java:4223) 
... 
at android.app.Activity.startActivityForResult(Activity.java:4182)
```

这是因为7.0执行了“StrictMode API 政策”。

_应对策略：使用FilrProvider来解决这一个问题_

####使用FileProvider

1. 在manifest里注册provider

    ```
    <provider 
        android:name="android.support.v4.content.FileProvider"
        android:authorities="com.jph.takephoto.fileprovider"
        android:grantUriPermissions="true"
        android:exported="false">
         <meta-data 
            android:name="android.support.FILE_PROVIDER_PATHS"
            android:resource="@xml/file_paths"/> 
    </provider>
    ```

    _exported必须要求为false，为true则会报安全异常。grantUriPermissions为true，表示授予URI临时访问权限。_

2. 指定共享目录

    为了指定共享的目录我们需要在资源目录下（res）创建一个xml目录，然后创建一个名为“file_paths”（名字可以随便起，只要和在manifest里注册的provider所引用的resource保持一致即可）的资源文件

    ```
    <?xml version="1.0" encoding="utf-8"?>
    <resources>
        <paths>
            <external-path path="" name="camera_photos" />
        </paths>
    </resources>
    ```
    
    * 代表的根目录： Context.getFilesDir()
    * 代表的根目录: Environment.getExternalStorageDirectory()
    * 代表的根目录: getCacheDir()

    _path=""是有意义的，它代表根目录，你可以向其他的应用共享根目录及其子目录下的任何一个文件。若设置path="pictures"，它代表着根目录下的pictures目录，那么你想向其他应用共享pictures目录范围之外的文件是不可行的。_

3. 使用FileProvider

    上述工作做完之后我们就可以使用FileProvider了，以调用相机为例：

    ```
    File file=new File(Environment.getExternalStorageDirectory(),
            "/temp/"+System.currentTimeMillis() + ".jpg");
    if (!file.getParentFile().exists()) {
        file.getParentFile().mkdirs();
    }

    Uri imageUri = FileProvider.getUriForFile(context,
        "com.jph.takephoto.fileprovider", file);//通过FileProvider创建一个content类型的Uri
    Intent intent = new Intent();
    intent.addFlags(Intent.FLAG_GRANT_READ_URI_PERMISSION); //添加这一句表示对目标应用临时授权该Uri所代表的文件
    intent.setAction(MediaStore.ACTION_IMAGE_CAPTURE);//设置Action为拍照
    intent.putExtra(MediaStore.EXTRA_OUTPUT, imageUri);//将拍取的照片保存到指定URI
    startActivityForResult(intent,1006);
    ```

    上面的代码有两处改变：

    1. 将之前Uri的scheme类型为file的Uri改成了有FileProvider创建一个content类型的Uri。
    2. 添加了intent.addFlags(Intent.FLAG_GRANT_READ_URI_PERMISSION);来对目标应用临时授权该Uri所代表的文件。

    _通过FileProvider的getUriForFile(Context context, String authority, File file)静态方法来获取URI，方法中的authority就是manifest里注册provider使用的authority。_

    _getUriForFile方法返回的Uri为：_

    ```
    content://com.jph.takephoto.fileprovider
            /camera_photos/temp/1474960080319.jpg`
    ```

    其中**camera_photos**就是file_paths文件中paths的name。

####裁剪照片

在7.0之前裁剪照片，使用以下方法：

```
File file=new File(Environment.getExternalStorageDirectory(),
        "/temp/"+System.currentTimeMillis() + ".jpg"); 
if (!file.getParentFile().exists()) {
    file.getParentFile().mkdirs();
}
Uri outputUri = Uri.fromFile(file);
Uri imageUri=Uri.fromFile(newFile("/storage/emulated/0/temp/1474960080319.jpg"));
Intent intent = new Intent("com.android.camera.action.CROP");
intent.setDataAndType(imageUri, "image/*"); 
intent.putExtra("crop", "true"); 
intent.putExtra("aspectX", 1); 
intent.putExtra("aspectY", 1); 
intent.putExtra("scale", true); 
intent.putExtra(MediaStore.EXTRA_OUTPUT, outputUri);
intent.putExtra("outputFormat", Bitmap.CompressFormat.JPEG.toString());
intent.putExtra("noFaceDetection", true); // no face detection
startActivityForResult(intent,1008);
```

和拍照一样，在7.0上同样会引起**FileUrlExposedException**，解决方法就是使用FileProvider。将上面的代码修改如下：

```
File file=new File(Environment.getExternalStorageDirectory(),
        "/temp/"+System.currentTimeMillis() + ".jpg");
if (!file.getParentFile().exists()) {
    file.getParentFile().mkdirs();
}
Uri outputUri = FileProvider.getUriForFile(context,
        "com.jph.takephoto.fileprovider",file); 
Uri imageUri=FileProvider.getUriForFile(context,"com.jph.takephoto.fileprovider",
        newFile("/storage/emulated/0/temp/1474960080319.jpg");//通过FileProvider创建一个content类型的Uri 
Intent intent = new Intent("com.android.camera.action.CROP");
intent.addFlags(Intent.FLAG_GRANT_READ_URI_PERMISSION);
intent.setDataAndType(imageUri, "image/*"); intent.putExtra("crop", "true");
intent.putExtra("aspectX", 1); intent.putExtra("aspectY", 1);
intent.putExtra("scale", true); intent.putExtra(MediaStore.EXTRA_OUTPUT,outputUri);
intent.putExtra("outputFormat", Bitmap.CompressFormat.JPEG.toString());
intent.putExtra("noFaceDetection", true); // no face detection
startActivityForResult(intent,1008);
```

使用开源库来进行照片的获取和裁剪，比如TakePhoto

###电池和内存

Android6.0引入了低电耗模式，7.0又在电池和内存上做了进一步的优化，来减少应用对电量的消耗和对内存的占用。这些优化带来的一些规则的变更可能会影响你的应用访问资源文件，以及你的应用通过特定的隐式Intent与其他应用互动的方式。

####低电耗模式

在低电耗模式下，当用户设备未插接电源、处于静止状态且屏幕关闭时，该模式会推迟 CPU 和网络活动，从而延长电池寿命。 Android7.0通过在设备未插接电源且屏幕关闭状态下、但不一定要处于静止状态（例如用户外出时把手持式设备装在口袋里）时应用部分 CPU 和网络限制，进一步增强了低电耗模式。

具体规则：

1. 当设备处于充电状态且屏幕已关闭一定时间后，设备会进入低电耗模式并应用第一部分限制： 关闭应用网络访问、推迟作业和同步。
2. 如果进入低电耗模式后设备处于静止状态达到一定时间，系统则会对PowerManager.WakeLock、AlarmManager闹铃、GPS和Wi-Fi扫描应用余下的低电耗模式限制。 无论是应用部分还是全部低电耗模式限制，系统都会唤醒设备以提供简短的维护时间窗口，在此窗口期间，应用程序可以访问网络并执行任何被推迟的作业/同步。

####后台优化

Android中有一些隐式广播，使用这些隐式广播可以做一些特定的功能，如，当手机网络变成WiFi时自动下载更新包等。 但，这些隐式广播会在后台频繁启动已注册侦听这些广播的应用，从而带来很大的电量消耗，为缓解这一问题来提升设备性能和用户体验，在Android 7.0中**删除了三项隐式广播**，以帮助优化内存使用和电量消耗。

* 在Android 7.0上 应用不会收到**CONNECTIVITY_ACTION**广播，即使你在manifest清单文件中设置了请求接受这些事件的通知。但，在前台运行的应用如果使用BroadcastReceiver请求接收通知，则仍可以在主线程中侦听**CONNECTIVITY_CHANGE**。
* 在 Android 7.0上应用无法发送或接收**ACTION_NEW_PICTURE**或**ACTION_NEW_VIDEO**类型的广播。

_应对策略：Android框架提供多个解决方案来缓解对这些隐式广播的需求。例如，**JobScheduler API**提供了一个稳健可靠的机制来安排满足指定条件（例如连入无限流量网络）时所执行的网络操作。甚至可以使用**JobScheduler API**来适应内容提供程序变化。_

###其他更新

####性能改进，系统大提速

Android N操作系统代码量减少50%、全新JIT编译器安装速度提升75%，并减少50%应用程序代码，且应用运行速度提升幅度高达600%。

为开发者提供了Android Studio 2.2开发工具，其拥有更强大的UI设计工具，增强了对Java8和C++语言的支持，且得益于新JIT编译器，模块化应用的构建更加快捷。

####支持Vulkan API

Vulkan API可以说是Open GL的继任者，据了解，它可以为Android带来更好的图像效果，并且大幅度减少对CPU的占用，从而实现高效率低功耗。

####后台机制改进

在Android N最新版中，“最近使用的程序”界面完全重做。

为了实现提升效率的目标，Android N中多任务界面最多显示7个应用程序，也就是说，同时**最多只能有7个应用程序在前台执行**。

同时，Android N中“任务”按键也被赋予一个全新功能--**Quick Switch（快速切换）**，当用户双击“任务”按钮的时候，可以快速在当前和最后使用的应用中快速切换。

此外，Android N中还添加了**原生的一键清理按钮**，用户可以快速关闭所有已启动的程序。对于较长时间不活跃的应用，系统会自动清除。

####原生分屏多任务

除了后台机制改进，Android N还提供了原生的分屏多任务功能。该功能无需多言，各大OEM厂商早已稔熟于心。

####原生虚拟现实VR

ndroid N中添加了**原生的VR功能**，但是用户必须要搭配谷歌自家的VR平台**“Daydream”**使用。并且从今年秋季开始，各大Android厂商都将开始适配这项功能。

####通知栏功能强化

通知栏是每一代Android系统必备的更新。在Android N中下拉通知栏将**加入多个快捷开关**，并且新的通知栏将**支持显示更多信息**，并且用户可以**在通知栏中快速回复**。虽然这些功能在第三方ROM中早已支持，但是谷歌为了更好的推行，直接将其提供了Android N原生支持。


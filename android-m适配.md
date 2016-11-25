##Android 6.0权限适配

Android 6.0最大的变化就是权限的申请，6.0以下时，我们的app需要什么权限，就在manifest里申请，使用`<user-permission>`。在安装app的时候，根据权限声明显示一个权限列表，用户只有同意的权限才可以安装app。而在6.0以后，我们可以直接安装，当app需要我们授予不恰当的权限的时候，用户可以拒绝，比如app需要访问通讯录，我们可以拒绝这个权限。另外我们可以到设置页面，具体的应用下面的权限选项里查看和更改应用的权限。Android 6.0新的权限机制很好的保护了用户的隐私数据。

新的规则把权限分为两种类型
* Normal Permission 这类权限一般不涉及用户隐私，是不需要用户主动授予的，比如手机振动，访问网络等。这类权限有：
* Dangerous Permission 这类权限一般都涉及用户隐私，需要用户手动授权，比如访问相册，访问通讯录等。

普通权限有：

```
ACCESS_LOCATION_EXTRA_COMMANDS
ACCESS_NETWORK_STATE
ACCESS_NOTIFICATION_POLICY
ACCESS_WIFI_STATE
BLUETOOTH
BLUETOOTH_ADMIN
BROADCAST_STICKY
CHANGE_NETWORK_STATE
CHANGE_WIFI_MULTICAST_STATE
CHANGE_WIFI_STATE
DISABLE_KEYGUARD
EXPAND_STATUS_BAR
GET_PACKAGE_SIZE
INSTALL_SHORTCUT
INTERNET
KILL_BACKGROUND_PROCESSES
MODIFY_AUDIO_SETTINGS
NFC
READ_SYNC_SETTINGS
READ_SYNC_STATS
RECEIVE_BOOT_COMPLETED
REORDER_TASKS
REQUEST_INSTALL_PACKAGES
SET_ALARM SET_TIME_ZONE
SET_WALLPAPER
SET_WALLPAPER_HINTS
TRANSMIT_IR
UNINSTALL_SHORTCUT
USE_FINGERPRINT
VIBRATE WAKE_LOCK
WRITE_SYNC_SETTINGS

```

危险权限有：

```
group:android.permission-group.CONTACTS
permission:android.permission.WRITE_CONTACTS
permission:android.permission.GET_ACCOUNTS
permission:android.permission.READ_CONTACTS

group:android.permission-group.PHONE
permission:android.permission.READ_CALL_LOG
permission:android.permission.READ_PHONE_STATE
permission:android.permission.CALL_PHONE
permission:android.permission.WRITE_CALL_LOG
permission:android.permission.USE_SIP
permission:android.permission.PROCESS_OUTGOING_CALLS
permission:com.android.voicemail.permission.ADD_VOICEMAIL

group:android.permission-group.CALENDAR
permission:android.permission.READ_CALENDAR
permission:android.permission.WRITE_CALENDAR

group:android.permission-group.CAMERA
permission:android.permission.CAMERA

group:android.permission-group.SENSORS
permission:android.permission.BODY_SENSORS

group:android.permission-group.LOCATION
permission:android.permission.ACCESS_FINE_LOCATION
permission:android.permission.ACCESS_COARSE_LOCATION

group:android.permission-group.STORAGE
permission:android.permission.READ_EXTERNAL_STORAGE
permission:android.permission.WRITE_EXTERNAL_STORAGE

group:android.permission-group.MICROPHONE
permission:android.permission.RECORD_AUDIO

group:android.permission-group.SMS
permission:android.permission.READ_SMS
permission:android.permission.RECEIVE_WAP_PUSH
permission:android.permission.RECEIVE_MMS
permission:android.permission.RECEIVE_SMS
permission:android.permission.SEND_SMS
permission:android.permission.READ_CELL_BROADCASTS
```
我们看到危险权限都是分组的，这种分组的权限机制有什么用呢？

当我们在6.0以上的机器运行app的时候，当我们申请某个危险权限的时候，如果已经被用户授予了**同一组的某个危险的权限，那么系统会立即授予，不用再让用户手动点击同意**，比如我们申请`READ_CALENDAR`，如果用户之前已经同意过`WRITE_CALENDAR`，那么系统会直接授予。

在6.0以上，**权限是按组授予的**，某个权限组的某个权限被同意，那么整个组的权限就被授予了，组内其他权限在使用的时候系统直接会同意。

###相关API

1. 在manifest里添加需要的权限（普通权限）

2. 当我们需要使用某个危险权限的时候，先要检查是否有这个权限
```
checkSelfPermission(Activity activity,String permission)
```
    如果用户已经同意授予，返回的结果是**PackageManager.PERMISSION_GRANTED**

3. 如果返回的结果不是GRANTED就需要申请权限了，不过在这之前，我们需要判断是否还能展示权限对话框。因为申请权限对话框的时候里面有一个选项为**之后不再提醒**，如果用户勾选了这个选项，那么我们调用申请权限的方法也不会出现对话框了，这个之后我们只能提醒用户自己到设置里手动修改权限了。
```
shouldShowRequestPermissionRationale(Activity activity,String permission)
```
如果这个函数返回的是true，那么就表示用户没有勾选**不再提醒**的选项，申请权限对话框可以弹出，如果返回false，那么申请权限对话框就不再弹出，申请的结果一直都是之前的不同意。

**注意：当程序第一次申请这个权限的时候，这个方法返回的值是false。因为第一次申请权限弹出框来的时候使没有“不再提醒”的选项的！**

4. 如果检查对话框弹出返回的是true，那么我们就可以调用申请权限的方法了
```
requestPermissions(Activity activity,String[] permissions,int requestCode);
```
我们看到，申请权限是可以申请很多权限的，对话框是按照你传入的权限的顺序依次弹出的，用户需要给每个申请的权限做决定。

5. 处理申请权限的回调，复写Activity的
```
public void onRequestPermissionsResult(int requestCode,
        String permissions[], int[] grantResults) 
```
返回的方法里有你申请的权限数组，还有与之对应的用户授予结果数组。

###总结

按照上面的方法封装我们自己的权限代码，就可以实现适配6.0以上的目的。6.0还有其他的更新，不过对于开发来说，最重要的改变就是权限申请的更改了。
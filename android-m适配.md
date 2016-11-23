##Android 6.0权限适配

Android 6.0最大的变化就是权限的申请，6.0以下时，我们的app需要什么权限，就在manifest里申请，使用`<user-permission>`。在安装app的时候，根据权限声明显示一个权限列表，用户只有同意的权限才可以安装app。而在6.0以后，我们可以直接安装，当app需要我们授予不恰当的权限的时候，用户可以拒绝，比如app需要访问通讯录，我们可以拒绝这个权限。另外我们可以到设置页面，具体的应用下面的权限选项里查看和更改应用的权限。Android 6.0新的权限机制很好的保护了用户的隐私数据。

新的规则把权限分为两种类型
* Normal Permission 这类权限一般不涉及用户隐私，是不需要用户主动授予的，比如手机振动，访问网络等。这类权限有：

`
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
`
* Dangerous Permission 这类权限一般都涉及用户隐私，需要用户手动授权，比如访问相册，访问通讯录等。


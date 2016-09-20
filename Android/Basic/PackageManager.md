# Android包管理流程概述

#### 一、包管理简介

Android包管理(PackageManager)是系统Framework层的一个服务，用来管理应用的安装、卸载以及系统级的应用设置，并且提供已安装应用的信息查询功能。

安装和卸载是最常见的功能，本文主要介绍安装、卸载的流程，以及开机扫描和加载应用的流程。

应用的设置包含应用权限管理、应用偏好、启用禁用、隐藏显示等。

应用设置、权限、应用的名字图标以及组件等信息的查询，都可以在包管理中找到接口。

#### 二、应用安装和卸载

安装应用一般有两种方式：

1. 手机中，可以从各种应用市场和工具安装，也可以直接从APK文件安装，本质上都是安装APK文件。
2. 从电脑侧通过各种管理工具安装，本质上和研发调试一样，都是通过adb安装。

第一种情况，安装时一般是启动系统的PackageInstaller进行具体的安装动作，PackageInstaller会调用PackageManager进行安装，或者应用市场类应用直接调用PackageManager安装。

第二种使用的是adb工具，最终调用的是pm命令，pm命令是对PackageManager的一个封装。总之PackageManager是安装和卸载应用的实际执行者。

##### 2.1 安装/卸载API

应用的安装和卸载由系统提供的PackageInstaller实现，开发中很少使用。

一般应用需要安装或者卸载一个应用时，可以通过标准的Intent调起PackageInstaller进行操作，所有应用都可以进行这个操作。
```
Intent.ACTION_INSTALL_PACKAGE  "android.permission.REQUEST_INSTALL_PACKAGES"
Intent.ACTION_UNINSTALL_PACKAGE
```

PackageInstaller是通过PackageManager提供的以下两类接口，分别实现应用的安装和卸载。这些安装和卸载接口，都提供了Observer，给调用者监听进度和接收结果。
```
installPackage*() "android.permission.INSTALL_PACKAGES"
deletePackage() "android.permission.DELETE_PACKAGES"
```

应用的安装和卸载都要求对应的权限，这些权限必须是系统签名或者privileged应用才可以申请，所以一般的第三方应用市场不具备直接安装和卸载应用的权限。GMS的PlayStore能够直接安装应用是因为预置到了priv-app目录。

如果需要实现一个应用安装器，只需要申请到对应的权限，并调用PackageManager中的接口即可。Google就在GMS包中提供了GooglePackageInstaller来替代默认的PackageInstaller，以实现对要安装的应用进行安全检查等功能。

##### 2.2 安装/卸载

安装过程整体分为两大部分：

1. 复制APK到安装分区(data区或者SD卡)，初步解析应用信息。[copy.vsd](../../_attach/Android/copy.vsd)
2. 完全的解析APK：比对签名证书等身份信息、创建data目录、导出lib文件、应用授权、以及组件注册等详细信息扫描。[install.vsd](../../_attach/Android/install.vsd)

卸载过程比较简单，先删除在PackageManager中注册的各种信息，清理data目录、权限信息，最后删除APK、dex/oat等文件。[uninstall.vsd](../../_attach/Android/uninstall.vsd)

##### 2.3 应用开机加载

开机时SystemServer启动PackageManagerService，PackageManagerService构造函数中进行一系列初始化。其中包括：加载并优化共享库，扫描特定的APK安装目录对APK进行加载（加载类似APK安装过程，通过flags区分具体流程），加载完成后对加载到的应用设置信息进行保存。该过程结束后，SystemServer继续进行后续流程。

[boot_load.vsd](../../_attach/Android/boot_pms_load.vsd)

原生Android代码扫描的APK目录有：
```
/vendor/overlay
/system/framework
/system/priv-app
/system/app
/vendor/app
/oem/app
/data/app
/data/app-private
```

#### 三、应用设置

应用的动态设置关机时，被保存在data区(`/data/system`)，每次开机时再被加载到内存中，由PackageManagerService进行管理。

主要的文件有：
```
/data/system/packages.xml 所有包基本信息：包/权限列表/shared-user/签名等
/data/system/packages.list 包名/uid/data路径/selabel/gid
/data/system/users/0/runtime-permissions.xml 运行时权限设置
/data/system/users/0/package-restrictions.xml 禁用/应用偏好/隐藏
```
PackageManagerService维护一个对象mSettings(com.android.server.pm.Settings)，当发生应用安装、卸载和更新，或者用户通过系统界面修改设置后，修改都直接作用于此对象，此对象进行修改的保存。

##### 3.1 权限管理

如上文所述，应用的权限是在安装和卸载时更新，然后便保存在data区，每次开机加载。从Android M开始，Google添加了运行时授权，允许用户来管理隐私密切相关的权限。允许应用(targetSdkVersion>=23)在运行时再获取部分权限，而不是在安装时进行所有权限的授权。即把所有权限分为两类，运行时权限(RuntimePermissions)和非运行时权限，运行时权限支持运行时授权，其他的仍然在安装时授权。

对于低于SDK 23的应用，仍然默认安装时授权。即使是运行时权限，应用仍然需要在`AndroidManifest.xml`中声明，未声明的权限不能申请。运行时权限是针对每个用户单独设置的，不同用户的默认设置也可能不同，所以`runtime-permissions.xml`是保存在每个用户目录中。

在M中将运行时权限分为9大类（如下表），定义为`permission-group`，每个权限定义时可选的分配一个group，在设置中用一个开关项管理一个group中的所有权限。有关信息可以在`framework/base/core/res/AndroidManifest.xml`中查看，protectionLevel为dangerous的权限是运行时授权，权限组的定义和权限所属组的信息也在其中。

|权限组|说明|
|:----:|:----:|
|"android.permission-group.CONTACTS"|通讯录|
|"android.permission-group.CALENDAR"|日历|
|"android.permission-group.SMS"|短信|
|"android.permission-group.STORAGE"|存储空间|
|"android.permission-group.LOCATION"|位置信息|
|"android.permission-group.PHONE"|电话|
|"android.permission-group.MICROPHONE"|麦克风(录音)|
|"android.permission-group.CAMERA"|相机|
|"android.permission-group.SENSORS"|身体传感器|

如果应用要使用运行时授权功能，开发者需要在使用对应权限前进行检查，调用Context的checkPermission系列方法。如果检查结果是未授权，则应该调用Activity的requestPermissions方法要求授权，然后系统会调用PackageInstaller的界面进行授权。授权结果通过Activity的onRequestPermissionsResult回传，接收到结果后可以进行后续流程。权限设置是针对每个包的，同一个包内的不同组件不需多次请求。

系统应用的运行时权限，比如联系人的联系人权限，相机的相机权限等，都是第一次开机后默认授予的，不会要求用户确认。这个操作是在PackageManagerService中第一次开机后时进行的，通过DefaultPermissionGrantPolicy类中grantDefaultPermissions方法。而且系统应用也并不是所有的运行时权限都被授予，只有对应类型的应用必须或者常用的权限会被授予，这些都在DefaultPermissionGrantPolicy中有定义。

第三方应用(SDK >= 23)的运行时权限默认是全部禁止的。

每个人用户创建时也会通过DefaultPermissionGrantPolicy类中grantDefaultPermissions方法进行运行时权限的授权。

##### 3.2 应用偏好

PackageManager提供了修改默认浏览器的接口`setDefaultBrowserPackageName()`，以及`addPreferredActivity()`用来设置处理某种Intent的默认应用。这两个方法都要求系统签名的权限：`"android.permission.SET_PREFERRED_APPLICATIONS"`，一般是系统选择应用对话框ResolverActivity根据用户选择，调用这类方法进行保存。

PackageManager也有对应的get方法可以获取目前设置。

##### 3.3 禁用启用/隐藏显示

用户可以在应用设置中手动设置禁用系统应用，禁用掉的应用类似第三方应用的卸载，不会显示到应用列表，选择应用对话框等界面都不会显示，但是应用数据仍然存在，启用后即可继续使用。

禁用和启用可以针对应用的的组件(`Activity/Service`等)，也可以针对整个应用。应用可以在`AndroidManifest.xml`中声明组件或者整个应用默认禁用或启用，通过组件和application的enabled属性进行设置。application的设置优先级高于组件的设置。

运行时可以通过`setComponentEnabledSetting/setApplicationEnabledSetting`分别设置组件和应用的启用、禁用。但是调用两个方法修改其他应用的设置时，需要只有系统签名和privileged应用具有的权限：`"android.permission.CHANGE_COMPONENT_ENABLED_STATE"`。应用修改自己的设置没有限制。

隐藏/显示应用是供DeviceAdmin和多用户设置来显示/隐藏部分应用，限制当前用户的使用。功能通过`setApplicationHiddenSettingAsUser()`设置，需要系统签名和privileged应用才有的`"android.permission.MANAGE_USERS"`权限。 

#### 四、APP层接口

##### 4.1 查询应用信息

```
getPackage*()
getApplication*()
getActivity*()
getReceiver*()
getService*()
getProvider*()
...
```

##### 4.2 包变化广播

```
Intent.ACTION_PACKAGE_ADDED
Intent.ACTION_PACKAGE_REPLACED
Intent.ACTION_PACKAGE_REMOVED
Intent.ACTION_PACKAGE_CHANGED
...
```

##### 4.3 获取应用列表

```
query*()
getInstalledApplications()
getInstalledPackages()
getPackagesHoldingPermissions()
getHomeActivities()
...
```

##### 4.4 禁用/隐藏等设置

```
checkPermission()
getPermissionInfo()
grantRuntimePermission()
revokeRuntimePermission()
addPreferredActivity()
setDefaultBrowserPackageName()
setComponentEnabledSetting()
setApplicationEnabledSetting()
setApplicationHiddenSettingAsUser() 
...
```
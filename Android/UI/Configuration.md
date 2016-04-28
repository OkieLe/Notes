#### 一、Configuration简述

Configuration类(android.content.res.Configuration)是专门用来描述手机的配置信息。这些配置信息包括用户设置的配置项，如字体大小、语言等，也包括设备配置，如输入模式、屏幕尺寸等。这些配置项会影响到应用如何引用资源。

Configuration类中使用以下公开(public)变量保存各种配置信息，下标列出了变量表示的配置信息，以及与变量相关的资源限定符。

|数据类型|名字|说明|资源限定符|
|:-----:|:-----:|-----|------|
|int|densityDpi|设备分辨率(点密度dpi)|Screen pixel density (dpi)|
|float|fontScale|当前用户设置的字体缩放因子|×|
|int|hardKeyboardHidden|硬键盘是否隐藏的标志位|Keyboard availability|
|int|keyboard|设备连接的键盘类型|Primary text input method|
|int|keyboardHidden|是否有(任何类型的)键盘可见标志位|Keyboard availability|
|Locale|locale|当前用户设置的语言(资源限定符)|locale|
|int|mcc|IMSI MCC，移动信号国家码|mcc|
|int|mnc|IMSI MNC，移动信号网络码|mnc|
|int|navigation|设备上方向控制键盘类型|Primary non-touch navigation method|
|int|navigationHidden|是否有方向控制键盘可见标志位|Navigation key availability|
|int|orientation|屏幕整体方向|Screen orientation|
|int|screenHeightDp|屏幕可见区域的当前高度(单位dp)|Screen height|
|int|screenLayout|屏幕整体布局的bit mask|Screen size、Screen aspect、Layout Direction|
|int|screenWidthDp|屏幕可见区域的当前宽度(单位dp)|Screen width|
|int|smallestScreenWidthDp|应用正常操作中的最小屏幕宽度|Smallest screen width|
|int|touchscreen|设备触屏类型|Touchscreen type|
|int|uiMode|用户界面模式的bit mask|UI mode、Night mode|

*注*：上表基于Android4.4，API 19

变量与资源限定符不是一一对应关系，但是所有限定符都有一个对应的配置项存储资源匹配时需要的信息。fontScale本身不会影响资源的引用，没有对应的资源限定符。

#### 二、Configuration机制

##### 2.1 使用Configuration

代码中有时需要从Configuration中读取相关的数据，比如本地语言、屏幕方向等配置，可以通过以下方式获取。

1. 程序中可调用Activity的如下方法来获取Configuration对象
```Java
Configuration cfg = getResources().getConfiguration();
```
2. 也可以使用`ActivityManagerNative.getDefault()`获取IActivityManager对象，然后调用IActivityManager对象的getConfiguration方法获得Configuratin：
```Java
IActivityManager am = ActivityManagerNative.getDefault();
Configuration cfg = am.getConfiguration();
```

如果需要修改Configuratin中某些配置项，可以先通过上面两种方法获取当前默认配置项，然后修改需要改变的数据，然后调用IActivityManager对象的updateConfiguration方法应用改变。

修改配置项的必须声明`”android.permission.CHANGE_CONFIGURATION”`权限。下面是系统设置修改本地语言的代码。
```Java
IActivityManager am = ActivityManagerNative.getDefault();
Configuration config = am.getConfiguration();
config.setLocale(locale);
am.updateConfiguration(config);
```

还有另一个修改配置的接口updatePersistentConfiguration，设置里面字体大小的修改调用这个方法，两个方法的区别稍后说明。

##### 2.2 Configuration工作机制

**1. Configuration初始化**
Configuration是作为ActivityManagerService的成员(mConfiguration)存在，SystemServer启动时，在构造函数中初始化为默认值，设置的绝大多数属性的默认值是无效值，fontScale默认为1，然后以当前语言设置语言属性(locale)。

然后在SystemServer主线程里WindowManagerService.systemReady之后，用下面代码主动更新mConfiguration，更新的内容包括屏幕density、size、orientation、layout属性、touchscreen、keyboard、navigation等几乎所有设备硬件相关的属性，完成Configuration的初始化。
```Java
Configuration config = wm.computeNewConfiguration();
DisplayMetrics metrics = new DisplayMetrics();
WindowManager w = (WindowManager) context.getSystemService(Context.WINDOW_SERVICE);
w.getDefaultDisplay().getMetrics(metrics);
context.getResources().updateConfiguration(config, metrics);
```
**2. Configuration如何在应用中生效**
在应用进程启动时，ActivityManagerService将mConfiguration作为handleBindApplication()的参数传入ActivityThread，ActivityThread之后会构造一个ResourcesManager用来管理应用中的Resources，mConfiguration也会相应的传递到Resources里。所以在应用从Resources里获取资源的时候，Configuration就会发挥作用，具体的代码流程可以查看对应类的源代码。

同时也可以看到2.1两种不同方式取到的Configuration不是同一个对象，第一种方式取到的是应用进程内的，在应用初始化流程中可能有些值发生变化。第二个是ActivityManagerService中的对象，像是全局的属性，两个好像不是完全相同的。

接下来看Configuration的更新流程，Configuration变化后，要调用ActivityManager的updateConfiguration/updatePersistentConfiguration方法，两者的区别在于，对应变化是否要保存到Settings数据库，如果要保存，就会把配置写入数据库，系统原生的只有字体大小保存到数据库。从之前的示例代码可以看到语言变化是没有保存的。

更新时会看到ActivityManagerService和应用里的Configuration总是保持一致。如果系统设置变化，系统会通知前台以及后台缓存的所有进程，这一步会调用到Activity的onConfigurationChanged()。同时系统还会发送所谓的ACTION_CONFIGURATION_CHANGED广播，通知希望自己处理配置变化的应用。

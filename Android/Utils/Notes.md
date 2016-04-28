#### JIT

/build/core/combo/TARGET_linux-arm.mk中
```Makefile
# Enable the Dalvik JIT compiler if not already specified.
ifeq ($(strip $(WITH_JIT)),)
    WITH_JIT := true
endif
```

默认情况下，JIT是打开的，这里就是JIT的总开关。如果要关闭，可以在这段代码前面加上WITH_JIT := false

WITH_JIT控制JIT是否打开时，会影响到两个库：libandroid_runtime.so和libdvm.so，控制代码分别在/frameworks/base/core/jni/Android.mk、/dalvik/vm/Android.mk和/dalvik/vm/Dvm.mk这三个Makefile中，可以在这里搜索一个WITH_JIT查看一下。

如何觉得上面这个比较暴力，也可以用温柔一点儿的方法：
修改/system/build.prop
在最后一行添加:
```Makefile
dalvik.vm.execution-mode=int:jit #（开启JIT）
dalvik.vm.execution-mode=int:fast #（关闭JIT）
```
这样的话，APK是稳定了，JIT的速度优势却也没了，毕竟出问题的APK只是一小部分。我这里最后发包时还是打开了，让应用程序开发者在APK里去关闭吧。

只要在APK的AndroidManifest.xml中把<application>的Android:vmSafeMode属性设置true就可以对APK禁用JIT。

#### ADB

```shell
adb root
adb shell su -c setenforce 0
adb shell setprop debug.sf.dump.enable true
adb shell stop
adb shell start
adb shell setprop debug.sf.dump.png 0
#在此处重现复现问题，由于截图可能系统比较卡
adb shell setprop debug.sf.dump.png 50
#停止抓图
adb shell setprop debug.sf.dump.png 0
```

抓图文件保存在：
>Layers are dumped in the timestamped location and format:
`/data/sfdump.png<YYYY><MM><DD>.<HH><MM><SS>/sfdump<dump frame no.>_layer<layer no.>.png`

`adb shell getprop ro.sf.lcd_density`获取分辨率。

`getprop | busybox grep opr`查询当前SIM卡及网络类型。

#### Prop

在android系统中，有一些初始化的配置文件，例如:
```
/init.rc
/default.prop
/system/build.prop
```
文件里面里面配置了开机设置的系统属性值，这些属性值，可以通过getprop获取，setprop设置

获取指定key的配置值
```
getprop [key]
```
，如果不带参数，只是getprop则是显示系统所有的配置值。
```
[dalvik.vm.heapsize]: [24m]
[curlockscreen]: [1]
[ro.sf.hwrotation]: [0]
[ro.config.notification_sound]: [OnTheHunt.ogg]
[ro.config.alarm_alert]: [Alarm_Classic.ogg]
```
设置指定key的属性值
```
setprop [key] [value]
```

**watchprops**

监听系统属性的变化，如果期间系统的属性发生变化则把变化的值显示出来
```
/system # watchprops
1307501833 sys.settings_system_version = '37'
1307501836 sys.settings_system_version = '38'
1307501862 persist.sys.timezone = 'Asia/Hong_Kong'
```
其实这三个命令都是toolbox的子命令，如果有兴趣的可以看在android源码中看到其对应的源码：
`system/core/toolbox/`

**dropbox**

用于管理系统运行的一些日志文件，多是系统或某应用出错时的信息。默认情况下保存3天，5MB，1000个异常信息。

用于分析user无log情况下异常，有帮助。保存在/data/system/dropbox下，user版本adb无法直接获取，可以`dumpsys dropbox --print`用于输出信息。
例如：
```
dumpsys dropbox --print
Drop box contents: 9 entries
```

Android源代码工程目录下，进入到system/core/rootdir目录，里面有一个名为ueventd.rc文件，往里面添加一行：
```
/dev/hello 0666 root root
```
将/dev/hello文件编译为root权限。

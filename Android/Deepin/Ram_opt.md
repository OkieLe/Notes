**内存占用分析**

####一、利用Android API函数查看

#####1. ActivityManager查看可用内存。
```Java
ActivityManager.MemoryInfo outInfo = new ActivityManager.MemoryInfo();
am.getMemoryInfo(outInfo);
//outInfo.availMem即为可用空闲内存
```
#####2. android.os.Debug查询PSS，VSS，USS等单个进程使用内存信息
```Java
MemoryInfo[] memoryInfoArray = am.getProcessMemoryInfo(pids);
MemoryInfo pidMemoryInfo=memoryInfoArray[0];
pidMemoryInfo.getTotalPrivateDirty();
//getTotalPrivateDirty(): Return total private dirty memory usage in kB. USS
//getTotalPss(): Return total PSS memory usage in kB. PSS
//getTotalSharedDirty(): Return total shared dirty memory usage in kB. RSS
```

####二、直接对Android文件进行解析查询

/proc/cpuinfo系统CPU的类型等多种信息。

/proc/meminfo 系统内存使用信息
```
/proc/meminfo
MemTotal: 16344972 kB
MemFree: 13634064 kB
Buffers: 3656 kB
Cached: 1195708 kB
```
我们查看机器内存时，会发现MemFree的值很小。这主要是因为，在linux中有这么一种思想，内存不用白不用，因此它尽可能的cache和buffer一些数据，以方便下次使用。但实际上这些内存也是可以立刻拿来使用的。所以`空闲内存=free+buffers+cached=total-used`

通过读取文件/proc/meminfo的信息获取Memory的总量。
`ActivityManager. getMemoryInfo(ActivityManager.MemoryInfo)`获取当前的可用Memory量。

####三、通过Android系统提供的Runtime类，执行adb 命令(top,procrank,ps...等命令)查询

通过对执行结果的标准控制台输出进行解析。这样大大的扩展了Android查询功能.例如：
```Java
final Process m_process = Runtime.getRuntime().exec("/system/bin/top -n 1");
final StringBuilder sbread = new StringBuilder();
BufferedReader bufferedReader = new BufferedReader(new InputStreamReader(m_process.getInputStream()), 8192);
# procrank
Runtime.getRuntime().exec("/system/xbin/procrank");
```
内存耗用：VSS/RSS/PSS/USS
>Terms
- VSS - Virtual Set Size 虚拟耗用内存（包含共享库占用的内存）
- RSS - Resident Set Size 实际使用物理内存（包含共享库占用的内存）
- PSS - Proportional Set Size 实际使用的物理内存（比例分配共享库，占用的内存）
- USS - Unique Set Size 进程独自占用的物理内存（不包含共享库占用的内存）

>一般来说内存占用大小有如下规律：VSS >= RSS >= PSS >= USS

>procrank一般只有eng版本可以使用。

>对于支持python的环境，还可以使用smem工具进行分析。

####内存泄露

前提：内存泄露测试，需要使用ENG版本。

1. adb 模式下输入一下命令，当看到如下System进程内存>60M时候就认为有泄漏。
```
C:\Documents and Settings\Administrator>adb shell
root@HWY535D-C00:/ # dumpsys meminfo | grep 'system'
dumpsys meminfo | grep 'system'
    64696 kB: system (pid 869)
    18454 kB: com.android.systemui (pid 956)
               64696 kB: system (pid 869)
               18454 kB: com.android.systemui (pid 956)
```
2. 在adb模式下输入如下命令，当内存剩余（MemFree+Buffers+Cached）<50M，时候也可以认为是有内存泄露。
```
root@HWY535D-C00:/ # cat proc/meminfo
cat proc/meminfo
MemTotal:         419280 kB
MemFree:           10492 kB
Buffers:           14960 kB
Cached:            85152 kB
```
3. adb模式下查看系统内存页面状态，也就是不同大小的连续页面个数，可以看到内存碎片状况。
```
root@HWY535D-C00:/ # cat proc/buddyinfo
```

####内存优化方案

#####确定机器目前的内存现状。

- 机器总内存：
- 硬件设备占用的静态内存：
- 系统可用内存总和（MemTotal）：
- 开机剩余内存(Free+Buffer+Cache):
- 常驻应用占用内存:

抠电池重启机器，5分钟后检测以上信息，并记录。

*备注*：最好每个版本都记录相关的内存信息，确保代码、BUG等的修改不会对内存产生大的影响。

#####分析硬件设变占用的静态内存是否可以优化

协调硬件同事一起分析协议栈、modem等占用的静态物理内存是否可以优化。

#####检查系统服务、去掉没必要启动的服务

检查系统开机启动的系统服务，确定哪些服务没必要启动、可以去掉。例如，DRM等服务。

#####检查开机启动的应用，去掉没必要的开机启动应用

检查开机启动应用，确认哪些应用可以不需要开机启动，哪些应用开机启动之后可以自杀掉。主要需要对公司自研的APK、平台提供商的APK加以确认。

#####检查系统预置资源、去掉没用的系统资源

检查预置资源，将常用资源预置，同时减少系统中不使用的资源图片，对默认集成资源进行必要的筛查

#####预加载常用的类

可以对系统中常用的类进行预加载。

#####不集成动态壁纸

不集成系统原生或者平台自带的动态壁纸。动态壁纸对内存的使用较多，而且不易回收，建议不要在版本中集成。

#####低端手机可以考虑图片从RGB888该为RGB565

低端手机可以将图片从RGB888修改为565，去掉图片的透明效果。这样只有在显示高清图片时候，效果会稍差，能降低系统和应用进程的内存占用。这个之前测试效果非常理想。但是图片的效果也确实适合硬伤。。。这个需要权衡。

#####根据分辨率等、合理调整heapsize值，调整内存回收机制

可以根据分辨率，修正虚拟机堆栈的配置大小，调整相关的minfree、adj的阀值，保证虚拟机能正常使用，同事也不用耗用很大的内存。

#####低端手机可以合理减少手机后台应用数量

低端手机可以减少手机的后台应用数，这样有利于缓解内存压力。默认后台应用数为24，可以考虑降低到15 或者更少。

#####使用ZRAM内存压缩技术。（考虑到手机性能Swap 虚拟内存最好不要使用）

ZRAM类的内存压缩技术是通过在内存上换出一块区域做2:1的压缩和解压。这样做的有点是相当于再加了一块内存。但是缺点是这种压缩和解压本省会提高cpu的负荷，影响性能，改技术适合CPU功能加强而内存较少的项目。例如4核512M的这种配置，可以使用。但是需要调整相关参数。

#####使用KSM内存合并技术

KSM(Kernel Samepage Merging)  允许这个系统管理程序通过合并内存页面来增加并发虚拟机的数量。从而降低应用对内存的使用，提高内存剩余、

#####还有某些极端措施、例如常驻应用定期清理、凌晨杀等。可以在手机内存压力极大地时候实施

这个是在上家公司时候想到的内存优化策略。可以在凌晨2，3点，当用户不使用手机时候，或者手机内存较低的时候，可以对手机进行偷偷的重启。是手机经常保持在一个良好的状态下运行。

#####系统内置应用的优化

需要对Phone、SystemUI等常驻应用做内存优化，优化相关逻辑和资源，降低系统的冗余。避免出现常驻应用的内存泄露。

#####检查消灭系统中的异常、出错问题。去掉不必要的日志，尤其是反复打印的日志

消灭异常，出错、干掉不必要的日志，尤其是反复打的日志，提高系统效率。

#####其他方案

还有些其他方案。可以借鉴android高版本的内存优化。也可以参考其他公司、或者平台的优化策略。

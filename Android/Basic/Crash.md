# Backtrace分析

## 1.  Java Backtrace


从Java Backtrace，我们可以知道当时Process 的虚拟机执行状态。 Java Backtrace 依靠SignalCatcher 来抓取.
Google默认：SignalCatcher catchs SIGQUIT(3), and then print the java backtrace to /data/anr/trace.txt
MTK加强： SignalCatcher catchs SIGSTKFLT(16), and then print the java backtrace to /data/anr/mtktrace.txt( After 6577.SP/ICS2.MP)
修改系统属性`dalvik.vm.stack-trace-file`改变默认位置，原生为`/data/anr/traces.txt`


### 1.1 抓取的方式

- 在ENG Build 中

```
adb remount
adb shell chmod 0777 data/anr
adb shell kill -3 pid
adb pull /data/anr
```

- 在User Build 中

没有root 权限的情况下，只能直接pull已经存在的backtrace.
```
adb pull /data/anr
```

- 你可以尝试直接使用下面的脚本一次性抓取

```
adb remount
adb shell chmod 0777 data/anr
adb shell ps

@echo off
set processid=
set /p processid=Please Input process id:

@echo on
adb shell kill -3 %processid%

@echo off
ping -n 8 127.0.0.1>nul

@echo on
adb pull data/anr/traces.txt trace-%processid%.txt

pause
```


### 1.2 JavaBacktrace 解析

下面是一小段system server 的java backtrace 的开始
```
----- pid 682 at 2014-07-30 18:04:53 -----
Cmd line: system_server

JNI: CheckJNI is off; workarounds are off; pins=4; globals=1484 (plus 50 weak)

DALVIK THREADS:
(mutexes: tll=0 tsl=0 tscl=0 ghl=0)

"main" prio=5 tid=1 NATIVE
  | group="main" sCount=1 dsCount=0 obj=0x4193fde0 self=0x418538f8
  | sysTid=682 nice=-2 sched=0/0 cgrp=apps handle=1074835940
  | state=S schedstat=( 47858718206 26265263191 44902 ) utm=4074 stm=711 core=0
  at android.os.MessageQueue.nativePollOnce(Native Method)
  at android.os.MessageQueue.next(MessageQueue.java:138)
  at android.os.Looper.loop(Looper.java:150)
  at com.android.server.ServerThread.initAndLoop(SystemServer.java:1468)
  at com.android.server.SystemServer.main(SystemServer.java:1563)
  at java.lang.reflect.Method.invokeNative(Native Method)
  at java.lang.reflect.Method.invoke(Method.java:515)
  at com.android.internal.os.ZygoteInit$MethodAndArgsCaller.run(ZygoteInit.java:829)
  at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:645)
  at dalvik.system.NativeStart.main(Native Method)
```
我们一行一行来解析：

0. PID at Time  然后接着 Cmd line: process name
1. backtrace头: dvm thread ：“DALVIK THREADS:”
2. Global DVM mutex value: if 0 unlock,  else lock
    tll:  threadListLock, 
    tsl:  threadSuspendLock, 
    tscl: threadSuspendCountLock
    ghl:  gcHeapLock
    (mutexes: tll=0 tsl=0 tscl=0 ghl=0)
3. thread name, java thread Priority, [daemon], DVM thread id, DVM thread status.
    "main"  -> main thread -> activity thread
    prio  -> java thread priority default is 5, (nice =0, linux thread priority 120),  domain is [1,10]
    DVM thread id, NOT linux thread id
    DVM thread Status:
     ZOMBIE,  RUNNABLE, TIMED_WAIT, MONITOR, WAIT, INITALIZING,STARTING, NATIVE, VMWAIT, SUSPENDED,UNKNOWN
    "main" prio=5 tid=1 NATIVE
4. DVM thread status 
    group: default is “main”
      Compiler,JDWP,Signal Catcher,GC, FinalizerWatchdogDaemon, FinalizerDaemon, ReferenceQueueDaemon are system group
    sCount:  thread suspend count
    dsCount: thread dbg suspend count
    obj:     thread obj address
    Sef:  thread point address
    group="main" sCount=1 dsCount=0 obj=0x4193fde0 self=0x418538f8
5. Linux thread status
    sysTId:  linux thread tid
    Nice: linux thread nice value
    sched: cgroup policy/gourp id
    cgrp:  c group  
    handle:  handle address

 sysTid=682 nice=-2 sched=0/0 cgrp=apps handle=1074835940
6. CPU Sched stat
    Schedstat (Run CPU Clock/ns, Wait CPU Clock/ns,  Slice times)
    utm: utime, user space time(jiffies)
    stm: stime, kernel space time(jiffies)
    Core now running in cpu.
   7.-more Call Stack

### 1.3 几种常见的java backtrace

1. ActivityThread 正常状态/ActivityThread Normal Case
2. Java Backtrace Monitor case：Synchronized Lock 等待同步锁.
3. 执行JNI code 未返回，状态是native

## 2. Native Backtrace

### 2.1 Native Backtrace 抓取方式

- 利用debuggerd 抓取

MTK 制作了一个利用debuggerd 抓取Native backtrace 的tool RTT(Runtime Trace), 对应的执行命令是:
```
rtt built timestamp (Apr 18 2014 15:36:21)
USAGE : rtt [-h] -f function -p pid [-t tid]
  -f funcion : current support functions:
                 bt  (Backtrace function)
  -p pid     : pid to trace
  -t tid     : tid to trace
  -n name    : process name to trace
  -h         : help menu
```

- 添加代码直接抓取

Google 默认提供了CallStack API, 请参考
```
system/core/include/libutils/CallStack.h 
system/core/libutils/CallStack.cpp
```
可快速打印单个线程的backtrace.

### 2.2 解析Native Backtrace

你可以使用GDB, 或者addr2line 等 tool 来解析抓回的Native Backtrace, 从而知道当时正在执行的native 代码。如addr2line 执行：
```
arm-linux-androideabi-addr2line -f -C -e symbols address
```

 ## 3. Kernel Backtrace

 ### 3.1 Kernel Backtrace 抓取方式

- 运行时抓取
   AEE/RTT 工具(MTK)

- Proc System
```shell
cat proc/pid/task/tid/stack
```

- Sysrq-trigger
```shell
adb shell cat proc/kmsg > kmsg.txt
adb shell "echo 8 > proc/sys/kernel/printk #修改printk loglevel
adb shell "echo t > /proc/sysrq-trigger" #打印所有的backtrace
adb shell "echo w > /proc/sysrq-trigger" #打印'-D' status 'D' 的 process
```

- KDB

 长按音量上下超过10s
```
 btp             <pid>                
 Display stack for process <pid>
 bta             [DRSTCZEUIMA]        
 Display stack all processes
 btc                                  
 Backtrace current process on each cpu
 btt             <vaddr>              
 Backtrace process given its struct task add
```

- 添加代码直接抓取
```c++
#include <linux/sched.h>
dump_stack(); // 当前thread
show_stack(task, NULL); // 其他thread
```

### 3.2 Process/Thread 状态

```
 "R (running)",  /*   0 */
 "S (sleeping)",  /*   1 */
 "D (disk sleep)", /*   2 */
 "T (stopped)",  /*   4 */
 "t (tracing stop)", /*   8 */
 "Z (zombie)",  /*  16 */
 "X (dead)",  /*  32 */
 "x (dead)",  /*  64 */
 "K (wakekill)",  /* 128 */
 "W (waking)",  /* 256 */
```
通常一般的Process 处于的状态都是S (sleeping), 而如果一旦发现处于如D (disk sleep), T (stopped), Z (zombie) 等就要认真审查.

## 4. 几种典型的异常情况

### 4.1 Deadlock

下面这个case 可以看到PowerManagerService, ActivityManager, WindowManager 相互之间发生deadlock.
```
"PowerManagerService" prio=5 tid=25 MONITOR
  | group="main" sCount=1 dsCount=0 obj=0x42bae270 self=0x6525d5c0
  | sysTid=913 nice=0 sched=0/0 cgrp=apps handle=1696964440
  | state=S schedstat=( 5088939411 10237027338 34016 ) utm=232 stm=276 core=2
  at com.android.server.am.ActivityManagerService.bindService(ActivityManagerService.java:~14079)
- waiting to lock <0x42aa0f78> (a com.android.server.am.ActivityManagerService) held by tid=16 (ActivityManager)
      at android.app.ContextImpl.bindServiceCommon(ContextImpl.java:1665)
      at android.app.ContextImpl.bindService(ContextImpl.java:1648)
      at com.android.server.power.PowerManagerService.bindSmartStandByService(PowerManagerService.java:4090)
      at com.android.server.power.PowerManagerService.handleSmartStandBySettingChangedLocked(PowerManagerService.java:4144)
      at com.android.server.power.PowerManagerService.access$5600(PowerManagerService.java:102)
      at com.android.server.power.PowerManagerService$SmartStandBySettingObserver.onChange(PowerManagerService.java:4132)
      at android.database.ContentObserver$NotificationRunnable.run(ContentObserver.java:181)
      at android.os.Handler.handleCallback(Handler.java:809)
      at android.os.Handler.dispatchMessage(Handler.java:102)
      at android.os.Looper.loop(Looper.java:139)
      at android.os.HandlerThread.run(HandlerThread.java:58)
      "ActivityManager" prio=5 tid=16 MONITOR
      | group="main" sCount=1 dsCount=0 obj=0x42aa0d08 self=0x649166b0
      | sysTid=902 nice=-2 sched=0/0 cgrp=apps handle=1687251744
      | state=S schedstat=( 39360881460 25703061063 69675 ) utm=1544 stm=2392 core=2
      at com.android.server.wm.WindowManagerService.setAppVisibility(WindowManagerService.java:~4783)
- waiting to lock <0x42d17590> (a java.util.HashMap) held by tid=12 (WindowManager)
      at com.android.server.am.ActivityStack.stopActivityLocked(ActivityStack.java:2432)
      at com.android.server.am.ActivityStackSupervisor.activityIdleInternalLocked(ActivityStackSupervisor.java:2103)
      at com.android.server.am.ActivityStackSupervisor$ActivityStackSupervisorHandler.activityIdleInternal(ActivityStackSupervisor.java:2914)
      at com.android.server.am.ActivityStackSupervisor$ActivityStackSupervisorHandler.handleMessage(ActivityStackSupervisor.java:2921)
      at android.os.Handler.dispatchMessage(Handler.java:110)
      at android.os.Looper.loop(Looper.java:147)
      at com.android.server.am.ActivityManagerService$AThread.run(ActivityManagerService.java:2112)
      "WindowManager" prio=5 tid=12 MONITOR
      | group="main" sCount=1 dsCount=0 obj=0x42a92550 self=0x6491dd48
      | sysTid=898 nice=-4 sched=0/0 cgrp=apps handle=1687201104
      | state=S schedstat=( 60734070955 41987172579 219755 ) utm=4659 stm=1414 core=1
      at com.android.server.power.PowerManagerService.setScreenBrightnessOverrideFromWindowManagerInternal(PowerManagerService.java:~3207)
- waiting to lock <0x42a95140> (a java.lang.Object) held by tid=25 (PowerManagerService)
      at com.android.server.power.PowerManagerService.setScreenBrightnessOverrideFromWindowManager(PowerManagerService.java:3196)
      at com.android.server.wm.WindowManagerService.performLayoutAndPlaceSurfacesLockedInner(WindowManagerService.java:9686)
      at com.android.server.wm.WindowManagerService.performLayoutAndPlaceSurfacesLockedLoop(WindowManagerService.java:8923)
      at com.android.server.wm.WindowManagerService.performLayoutAndPlaceSurfacesLocked(WindowManagerService.java:8879)
      at com.android.server.wm.WindowManagerService.access$500(WindowManagerService.java:170)
      at com.android.server.wm.WindowManagerService$H.handleMessage(WindowManagerService.java:7795)
      at android.os.Handler.dispatchMessage(Handler.java:110)
      at android.os.Looper.loop(Looper.java:147)
      at android.os.HandlerThread.run(HandlerThread.java:58)
```

### 4.2 执行JNI native code 后一去不复返

```
"main" prio=5 tid=1 NATIVE
  | group="main" sCount=1 dsCount=0 obj=0x41bb3d98 self=0x41ba2878
  | sysTid=814 nice=-2 sched=0/0 cgrp=apps handle=1074389380
  | state=D schedstat=( 22048890928 19526803112 32612 ) utm=1670 stm=534 core=0
  (native backtrace unavailable)
  at android.hardware.SystemSensorManager$BaseEventQueue.nativeDisableSensor(Native Method)
  at android.hardware.SystemSensorManager$BaseEventQueue.disableSensor(SystemSensorManager.java:399)
  at android.hardware.SystemSensorManager$BaseEventQueue.removeAllSensors(SystemSensorManager.java:325)
  at android.hardware.SystemSensorManager.unregisterListenerImpl(SystemSensorManager.java:194)
  at android.hardware.SensorManager.unregisterListener(SensorManager.java:560)
  at com.android.internal.policy.impl.WindowOrientationListener.disable(WindowOrientationListener.java:139)
  at com.android.internal.policy.impl.PhoneWindowManager.updateOrientationListenerLp(PhoneWindowManager.java:774)
  at com.android.internal.policy.impl.PhoneWindowManager.screenTurnedOff(PhoneWindowManager.java:4897)
  at com.android.server.power.Notifier.sendGoToSleepBroadcast(Notifier.java:518)
  at com.android.server.power.Notifier.sendNextBroadcast(Notifier.java:434)
  at com.android.server.power.Notifier.access$500(Notifier.java:63)
  at com.android.server.power.Notifier$NotifierHandler.handleMessage(Notifier.java:584)
  at android.os.Handler.dispatchMessage(Handler.java:110)
  at android.os.Looper.loop(Looper.java:193)
  at com.android.server.ServerThread.initAndLoop(SystemServer.java:1436)
  at com.android.server.SystemServer.main(SystemServer.java:1531)
  at java.lang.reflect.Method.invokeNative(Native Method)
  at java.lang.reflect.Method.invoke(Method.java:515)
  at com.android.internal.os.ZygoteInit$MethodAndArgsCaller.run(ZygoteInit.java:824)
  at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:640)
  at dalvik.system.NativeStart.main(Native Method)

===>
KERNEL SPACE BACKTRACE, sysTid=814
 [<ffffffff>] 0xffffffff from [<c07e5140>] __schedule+0x3fc/0x950
 [<c07e4d50>] __schedule+0xc/0x950 from [<c07e57e4>] schedule+0x40/0x80
 [<c07e57b0>] schedule+0xc/0x80 from [<c07e5ae4>] schedule_preempt_disabled+0x20/0x2c
 [<c07e5ad0>] schedule_preempt_disabled+0xc/0x2c from [<c07e3c3c>] mutex_lock_nested+0x1b8/0x560
 [<c07e3a90>] mutex_lock_nested+0xc/0x560 from [<c05667d8>] gsensor_operate+0x1bc/0x2c0
 [<c0566628>] gsensor_operate+0xc/0x2c0 from [<c0495fa0>] hwmsen_enable+0xa8/0x30c
 [<c0495f04>] hwmsen_enable+0xc/0x30c from [<c0496500>] hwmsen_unlocked_ioctl+0x2fc/0x528
 [<c0496210>] hwmsen_unlocked_ioctl+0xc/0x528 from [<c018ad98>] do_vfs_ioctl+0x94/0x5bc
 [<c018ad10>] do_vfs_ioctl+0xc/0x5bc from [<c018b33c>] sys_ioctl+0x7c/0x8c
 [<c018b2cc>] sys_ioctl+0xc/0x8c from [<c000e480>] ret_fast_syscall+0x0/0x40
 [<ffffffff>]  from [<ffffffff>] 
```

# 系统环境

## 1. 系统运行环境

客观的反应系统的执行环境，通常包括如CPU 利用率，Memory 使用情况， Storage 剩余情况等。这些资料也非常重要，比如可以快速的知道，当时是否有Process 在疯狂的执行，当时是不是处于严重的low memory 情况， Storage 是否有耗尽的情况发生等。

## 2. CPU Usage

追查CPU 利用率可大体的知道，当时机器是否有Process 在疯狂的运行,  当时系统运行是否繁忙。通常死机分析，只需要抓取基本的使用情况即可。下面说一下一般的抓取方式

### 2.1 top

top 可以简单的查询Cpu 的基本使用情况
```
Usage: top [ -m max_procs ] [ -n iterations ] [ -d delay ] [ -s sort_column ] [ -t ] [ -h ]
    -m num  Maximum number of processes to display.
    -n num  Updates to show before exiting.
    -d num  Seconds to wait between updates.
    -s col  Column to sort by (cpu,vss,rss,thr).
    -t      Show threads instead of processes.
    -h      Display this help screen.
```
注意的是top 的CPU% 是按全部CPU 来计算的，如果以单线程来计算，比如当时有开启4个核心，那么最多吃到25%. 

个人常见的操作方式：
```shell
top -t -m 5 -n 2
```
正常情况：
```
User 2%, System 12%, IOW 0%, IRQ 0%
User 14 + Nice 0 + Sys 67 + Idle 471 + IOW 0 + IRQ 0 + SIRQ 2 = 554
  PID   TID PR CPU% S     VSS     RSS PCY UID      Thread          Proc
 2423  2423  1   8% R   2316K   1128K     root     top             top
  270   270  0   1% S   2160K    924K     root     aee_resmon      /system/bin/aee_resmon
  159   159  0   0% D      0K      0K     root     bat_thread_kthr
    3     3  0   0% S      0K      0K     root     ksoftirqd/0
   57    57  0   0% D      0K      0K     root     hps_main
 
User 1%, System 7%, IOW 0%, IRQ 0%
User 10 + Nice 0 + Sys 41 + Idle 494 + IOW 0 + IRQ 0 + SIRQ 0 = 545
  PID   TID PR CPU% S     VSS     RSS PCY UID      Thread          Proc
 2423  2423  1   8% R   2324K   1152K     root     top             top
   57    57  0   0% D      0K      0K     root     hps_main
  242   419  0   0% S   8600K   4540K     shell    mobile_log_d    /system/bin/mobile_log_d
  982   991  0   0% S   4364K   1156K     media_rw sdcard          /system/bin/sdcard
  272   272  0   0% S  30680K   9048K     root     em_svr          /system/bin/em_svr
```
从上面可以看出, 系统基本运行正常，没有很吃CPU 的进程.

异常情况：
```
User 59%, System 4%, IOW 2%, IRQ 0%
User 1428 + Nice 0 + Sys 110 + Idle 811 + IOW 67 + IRQ 0 + SIRQ 1 = 2417
  PID   TID PR CPU% S     VSS     RSS PCY UID      Thread          Proc
16132 32195  3  14% R 997100K  53492K  bg u0_a60   Thread-1401     com.android.mms
16132 32190  1  14% R 997100K  53492K  bg u0_a60   Thread-1400     com.android.mms
16132 32188  2  14% R 997100K  53492K  bg u0_a60   Thread-1399     com.android.mms
16132 32187  0  14% R 997100K  53492K  bg u0_a60   Thread-1398     com.android.mms
18793 18793  4   1% R   2068K   1020K     shell    top             top
 
User 67%, System 3%, IOW 7%, IRQ 0%
User 1391 + Nice 0 + Sys 75 + Idle 435 + IOW 146 + IRQ 0 + SIRQ 1 = 2048
  PID   TID PR CPU% S     VSS     RSS PCY UID      Thread          Proc
16132 32195  3  16% R 997100K  53492K  bg u0_a60   Thread-1401     com.android.mms
16132 32188  2  16% R 997100K  53492K  bg u0_a60   Thread-1399     com.android.mms
16132 32190  0  16% R 997100K  53492K  bg u0_a60   Thread-1400     com.android.mms
16132 32187  1  16% R 997100K  53492K  bg u0_a60   Thread-1398     com.android.mms
18793 18793  4   2% R   2196K   1284K     shell    top             top
```
可以明显的看到，mms 的4个thread 都有进入了deadloop, 分别占用了一个cpu core.  同时可以快速抓取他们的java trace,  进一步可以看到当时MMS 的四个backtrace, 以便快速分析.
```
"Thread-1401" prio=5 tid=32 SUSPENDED JIT
  | group="main" sCount=1 dsCount=0 obj=0x4264f860 self=0x7b183558
  | sysTid=32195 nice=0 sched=0/0 cgrp=apps/bg_non_interactive handle=2078705952
  | state=S schedstat=( 3284456714198 104216273858 383002 ) utm=324720 stm=3725 core=5
  at com.yulong.android.mms.c.f.d(MmsChatDataServer.java:~1095)
  at com.yulong.android.mms.ui.MmsChatActivity$37.run(MmsChatActivity.java:7582)
  at java.lang.Thread.run(Thread.java:841)
 
"Thread-1400" prio=5 tid=31 SUSPENDED JIT
  | group="main" sCount=1 dsCount=0 obj=0x41f5d8f0 self=0x7be2a8e8
  | sysTid=32190 nice=0 sched=0/0 cgrp=apps/bg_non_interactive handle=2078029504
  | state=S schedstat=( 3284905134412 105526230562 382946 ) utm=324805 stm=3685 core=5
  at com.yulong.android.mms.ui.MmsChatActivity$37.run(MmsChatActivity.java:~7586)
  at java.lang.Thread.run(Thread.java:841)
 
"Thread-1399" prio=5 tid=30 SUSPENDED JIT
  | group="main" sCount=1 dsCount=0 obj=0x42564d28 self=0x7b0e6838
  | sysTid=32188 nice=0 sched=0/0 cgrp=apps/bg_non_interactive handle=2077662640
  | state=S schedstat=( 3288042313685 103203810616 375959 ) utm=325143 stm=3661 core=7
  at com.yulong.android.mms.ui.MmsChatActivity$37.run(MmsChatActivity.java:~7586)
  at java.lang.Thread.run(Thread.java:841)
 
"Thread-1398" prio=5 tid=29 SUSPENDED
  | group="main" sCount=1 dsCount=0 obj=0x4248e5a8 self=0x7be0d128
  | sysTid=32187 nice=0 sched=0/0 cgrp=apps/bg_non_interactive handle=2079251904
  | state=S schedstat=( 3287248372432 105116936413 379634 ) utm=325055 stm=3669 core=6
  at com.yulong.android.mms.ui.MmsChatActivity$37.run(MmsChatActivity.java:~7586)
  at java.lang.Thread.run(Thread.java:841)
```
当时处于suspend, 即意味着当时这四个thread 正在执行java code, 而抓取backtrace 时强制将thread suspend。需要review 代码设计。

### 2.2 cputime

通常用cputime 来打印一段时间内, CPU 的利用率的统计情况, 资讯比top 更加详细.
```shell
cputime -h
Usage: cputime  [-start/-stop]  [-n iterations] [-d delay] [-e time] [ -m max_count ]  [-p] [-s sort_column] [-i id] [-h]
 -start       Start cpu time monitor.
 -stop        Stop cpu time monitor.
 -n  num      Updates to show before exiting.
 -d  num      Seconds to wait between updates.
 -m  num      Maximum number of information to display.
 -e  num      Enable CPU time monitor and stop monitor after num seconds. If no this parameter will
              show last cputime monitor data.
 -p           Show process instead of thread. If no this parameter default will show thread information.
 -s  col      Column to sort by time/user/kernel/id/isr_c/isr_t(cputime/user time/kernel time/id/isr_count/
              isr_time). If no this parameter default will sort by cputime.
 -i  id       show isr information of thread id.
 -h           Display this help screen.

****** Example ******
cputime -e 100: Enable cputime monitor and stop after 100 seconds. Then show thread cputime, and sort by cputime.
cputime -e 200  -s user: Enable cputime monitor and stop after 200 seconds. Then show thread cputime, and sort by user time.
cputime: Show thread cputime, and sort by cputime.
cputime -p -s id: Show process cputime, and sort by process id.
```

### 2.3 ftrace

ftrace 可以纪录CPU 最为详细的执行情况, 即linux scheduler 的执行情况. 通常默认只开启 sched_switch. 
如何抓取ftrace可以查询相关的FAQ

### 2.4 Kernel core status

有的时候我们需要追查一下，当时Kernel 的基本调度情况，以及接收中断的情况，以判断当前CPU 执行的基本情况是否异常。比如有时候如果某个中断上来太过频繁，就容易导致系统运行缓慢，甚至死机。
```shell
* CPU Sched status
     adb shell cat proc/sched_debug
     Use sysrq-trigger
* CPU interrupts
     adb shell cat proc/interrupts
     adb shell cat proc/irq/irq_id/spurious
```

## 3. Memory Usage

Memory Usage, 我们通常会审查，系统当时memory 是否足够, 是否处于low memory 状态, 是否可能出现因无法申请到memory 而卡死的情况.

常见的一些基本信息如下:
```shell
* meminfo: basic memory status
adb shell cat proc/meminfo
adb shell cat proc/pid/maps
adb shell cat proc/pid/smaps
* procrank info: all process memory status
adb shell procrank
adb shell procmem pid
adb shell dumpsys meminfo pid
* zoneinfo:
adb shell cat proc/zoneinfo
* buddyinfo:
adb shell cat /proc/buddyinfo
```

## 4. Storage Usage

查看Storage 的情况，通常主要是查询data 分区是否已经刷满，sdcard 是否已经刷满，剩余的空间是否足够。以及是否有产生超大文件等。

通常使用的命令

- df
```shell
Filesystem               Size     Used     Free   Blksize
/dev                   446.0M   128.0K   445.8M   4096
/sys/fs/cgroup         446.0M    12.0K   445.9M   4096
/mnt/secure            446.0M     0.0K   446.0M   4096
/mnt/asec              446.0M     0.0K   446.0M   4096
/mnt/obb               446.0M     0.0K   446.0M   4096
/system                  1.2G   915.3M   355.5M   4096
/data                    1.1G   136.7M  1010.1M   4096
/cache                 106.2M    48.0K   106.2M   4096
/protect_f               4.8M    52.0K     4.8M   4096
/protect_s               4.8M    48.0K     4.8M   4096
/mnt/cd-rom              1.2M     1.2M     0.0K   2048
/mnt/media_rw/sdcard0     4.6G     1.1G     3.4G   32768
/mnt/secure/asec         4.6G     1.1G     3.4G   32768
/storage/sdcard0         4.6G     1.1G     3.4G   32768
```

- du
```shell
du -help
usage: du [-H | -L | -P] [-a | -d depth | -s] [-cgkmrx] [file ...]

du -LP -d 1
8       ./lost+found
88      ./local
384     ./misc
48      ./nativebenchmark
912     ./nativetest
8       ./dontpanic
13376   ./data
8       ./app-private
8       ./app-asec
129424  ./app-lib
8       ./app
136     ./property
16      ./ssh
116312  ./dalvik-cache
8       ./resource-cache
48      ./drm
8       ./security
32      ./sec
8       ./user
16      ./media
8       ./anr
32      ./mdlog
1312    ./system
176     ./recovery
32      ./backup
274688  .
```

# 进程环境

## 系统运行环境

当我们怀疑死机问题可能是某个进程出现问题而引发时，通常我们需要对这个进程进行深入的分析, 即进程运行环境分析。通常包括分析如，线程状态，各种变量值，寄存器状态等。在Android 系统中，我们将其划分成三个层次。
即 Java 运行环境分析, Native 运行环境分析, Kernel 运行环境分析. 下面分别说明.

## Java 运行环境分析

对于Zygote fork 出来的process, 如APP 以及system_server, 都会进行Java 运行环境分析. 其关键是分析Java Heap, 以便快速知道某个Java 变量的值, 以及Java 对象的分布和引用情况。

通常Java Heap 的分析方式则是抓取Java Hprof, 然后使用MAT 等工具进行分析.

### 抓取Hprof 的手法，如:

第一种方式: 使用am 命令
```
adb shell am dumpheap {Process} file

# EXAMPLE
adb shell chmod 777 /data/anr
adb shell am dumpheap com.android.phone /data/anr/phone.hprof

adb pull /data/anr/phone.hprof
```

第二种方式: 使用DDMS 命令

在DDMS 中选择对应的process, 然后在Devices 按钮栏中选择Dump Hprof file， 保存即可
 
第三种方式: 通过代码的方式

在android.os.Debug 这个class 中有定义相关的抓取hprof 的method.如：
```
public static void dumpHprofData(String fileName) throws IOException;
```
这样即可在代码中直接将这个process 的hprof 保存到相对应的文件中，注意这个只能抓取当时的process，如果想抓其他的process 的hprof, 那么就必须通过AMS 帮忙了。可以先获取IActivityManager 接口，然后调用它的dumpheap 方法。具体的代码，大家可以参考
`frameworks/base/cmds/am/src/com/android/commands/am/am.java`中的调用代码
 
第四种方式: 发送SIGUSER1

在部分机器中，如果具有root 权限，可以直接发送SIG 10 来抓取, 此时对应的Hprof 保存在/data/misc下面，文件名如：`heap-dump-tm1357153307-pid1882.hprof`

### 快速分析

首先, DVM 的Hprof 和 标准的Java Hprof 有一些差别, 需要使用hprof-conv 进行一次转换, 将DVM 格式的hprof 转换成标准的java 命令的hprof
```
hprof-conv in.hprof out.hprof
```
其次, 使用如MAT Tool, 打开转换后的hprof 文件, 通常我们会

> analysis  java thread information
> analysis  java var value
> analysis  Object reference
> analysis  GC path

具体如何使用MAT 分析可以参考MAT 的官方网站.
 
## Native 运行环境分析

Native 运行环境分析，我们通常会采用Core dump 分析手法. Core dump 纪录了当时进程的各类关键资讯，比如变量参数，线程stack, heap, register 等。通常可以认为是这个Process 当时最为完整的资讯了。 但Core dump 往往比较大, 有时甚至会超过1G, 属于比较重量型的分析手法了。

### 如何抓取Core Dump

目前MTK 的机器会将Core Dump 转换成AEE DB. 否则对应的Core dump 文件即存放在/data/core 目录下。手工抓取时, 可以：
```
adb shell aee -d coreon
adb shell kill -31 PID
```
此时core dump 就可能存放在两个目录下: /data/core, 以及 /sdcard/mtklog/aee_exp 下面新的DB 文件.

### 如何分析Core Dump

因为通常已经将Core Dump 转换成了AEE DB. 所以首先将AEE DB 解开, 即可以看到PROCESS_COREDUMP 的文件，有的时候此文件很大, 比如超过1G。而分析Core Dump 的Tools 很多, 比如traces32, GDB 等，这里就不详加说明，可以参考网络上的相关文档.
 
## Kernel 运行环境分析

ramdump功能，可以发生KE后将物理内存压缩保存到SD卡上，拿到该文件后就可以转换为kernel space，查看kernel各种变量，比查看kernel log更加方便快捷.

MTK只有在eng版本下支持该功能，并且是EMMC的，存在内置T卡才行,在projectConfig.mk里的MTK_SHARED_SDCARD必须为no即MTK_SHARED_SDCARD=no。

连上adb后：
```
adb shell
#echo Y > /sys/module/mrdump/parameters/enable
#echo emmc > /sys/module/mrdump/parameters/device
#echo sdcard > /sys/module/mrdump/parameters/device
```
这样就开启了ramdump功能，注意重启后无效，必须重新设置才行，之后重新开机，此时会在内置T卡或外置T卡看到CEDump.kdmp文件，结合`kernel/out/vmlinux`或`out/target/product/$proj/obj/KERNEL_OBJ/vmlinux`，一起提供给Mediatek即可做进一步kernel异常重启的分析.
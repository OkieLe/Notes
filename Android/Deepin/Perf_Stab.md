## 1.  ANR问题

1. 看CPU占用率、iowait占用率
2. 是否有死锁（held by/MONITOR）
3. 主线程被什么线程阻塞（held by）
4. 是否main县城Binder远程调用未返回
5. 是否线程太多卡顿
6. 哪些线程blocked
7. 看进程的binder状态

## 2.  常见signal原因

1. Signal 6：abort信号，通过反编译可以知道走到那个代码分支，通过逻辑判断。一般都是在程序中检查状态，代码中发送的信号。
2. Signal 11：访问地址错误，反编译分析代码，可能是地址重复释放，数据越界，释放完再使用，多线程操作数据异常。
3. Signal 9：用户触发或者程序发出的杀进程，比如lmk。
4. Signal 8：基本都是除以0造成的，找到地方好处理。
5. Signal 13：管道破裂，比如往关掉的socket写数据。可以在初始化的时候忽略此信号`signal(SIGPIPE, SIG_IGN)`。

## 3.  Watchdog原因

1. 持有重要服务对象太久不释放
2. 重要线程执行很复杂的操作
3. SystemServer线程死锁
4. CPU占用过高
5. Binder耗尽
6. 进程间通信Block
7. 驱动、modem异常，不能及时响应

## 4.  进程Crash原因

1. 空指针、数组越界、除0
2. 变量多线程操作异常，`ConcurrentModificationException`
3. 代码自己抛的异常
4. OOM
5. JNI引用超限，每个虚拟机51200(全局引用防止Java回收)，“global reference table overflow”。

## 5.  冻屏

界面有显示，但是不能响应任何触摸、按键操作，可能并没有ANR

1. 卡在界面，查看SystemServer服务是否都已启动完成，搜索“SystemServer”查看。
2. `adb shell getevent`，查看是否有操作对应的event，如果没有需要驱动分析。
3. `adb shell dumpsys input`，查看是否只有down事件，排除鬼手问题
4. `adb shell dumpsys window`，查看窗口焦点有无异常，当前焦点窗口有无异常
5. `adb shell am dumpheap PACKAGE sdcard/a.hprof`获取hprof文件，利用内存分析工具MAT分析当前进程状态
6. `adb shell ps`查看系统中进程状态是否异常
7. `adb shell kill –6 PID`生成trace.txt
8. 没有端口的可以考虑用OTG连上鼠标看是否可以接受点击

## 6.  稳定性问题日志系统整体设计方案

| 分类      | 问题细分             | 解决方案                                     | 技术点                                      |
| ------- | ---------------- | ---------------------------------------- | ---------------------------------------- |
| Linux内核 | 1 数据和指令预取异常      | Panic记录内核日志到apanic_console，记录线程调用栈信息到apanic_threads，扩大两个文件大小并添加其他状态信息，如zoneinfo、meminfo、slabinfo、vmalloc info、virtual memory stats、cpuinfo、vm traces、kernel cpufreq、processes&threads、fs等 | PanicOopsapanic                          |
|         | 2 内核死锁/死循环，可中断挂起 | 加入死锁检测，并用panic抓取调用栈信息用sysrq去收集内核个进程调用栈信息 | Panicsoft lockup detecthung task detectsysrq |
|         | 3 不可中断挂起         | 用trace导出内存数据                             | 内存数据分析                                   |
| JNI层    | 1 进程异常           | 在debuggerd中通过ptrace系统调用获取出现问题的进程的调用栈信息   | debuggerdptracetombstone                 |
|         | 2 进程崩溃           | 开启系统coredump，进程崩溃时到处coredump分析           | coredump                                 |
|         | 3 死锁/死循环         | 加入死锁和死循环检测，自动向死锁进程发送SIGSEGV信号，抓取调用栈信息，输出tombstone文件 | 死锁检测SysrqTombstone                       |
| Java应用  | 1 程序异常           | 监控异常，发现异常时调用bugreport输出各种状态信息            | DropboxBugreport                         |
|         | 2 mutex死锁/死循环    | Java虚拟机会杀死超过5s没响应的应用，并声称trace，包含调用栈信息    | Anr                                      |

## 7.  常见性能问题方案

1. 应用启动慢
   1. 启动提到最高频率，保持一段时间
   2. 调整AMS中Cache进程的比率，缓解常用应用的启动问题
   3. 常用应用，可常驻内存，禁止任务清理
   4. 短暂调整参数，使应用在大核上启动
2. 运行慢
   1. 固定场景提频（转屏、滑动）
   2. 进程冷冻，冻结后台进程
   3. 通过cgroup机制，提升前台进程优先级
   4. 改变CPU调度策略、门限
   5. 限制应用自启动，主动清理后台，减少多任务运行情况
   6. 调整参数，使应用在大核上运行
3. 帧率低
   1. CPU/DDR/GPU不限频率，甚至提频，加速Layout进程
   2. 开启部分应用的强制GPU绘制功能，减少Draw的时间
   3. 调整参数，使应用在大核上运行
4. 内存不足
   1. 某些耗内存大户启动后，主动清理后台
   2. 调整lmk参数，今早终结应用，释放内存。
5. 开机慢
   1. 开机应始终在大核上运行

## 8.  开机性能分析

除去利用log进行分析以外，可以用BootChart来分析。

BootChart是Linux中用来分析启动过程依赖的工具，可以用来分析依赖关系，提升并行度，从而提升启动性能。具体样例参考[www.bootchart.org](http://www.bootchart.org/) 的samples页面。

Android中，init进程解析init.rc时，data分区挂载后立即触发。

BootChart触发开关：

开始条件：存在`/data/bootchart/start`文件，切文件中包含有效的时长信息（单位秒），或者内核命令行参数中有`androidboot.bootchart=<时长>`时会触发日志记录。

停止条件：达到最大采样时长时停止，默认10分钟，或者`/data/bootchart/stop`中包含1时停止。

BootChart中记录的信息：

采样过程中从`/proc/stat`、`/proc/diskstats`、`/proc/<pid>`中读取数据分别写入`/data/bootchart`下面的日志文件中。

## 9. 内存管控

保存状态信息脚本：

```getlog.bat
echo %TIME% > mem.txt
adb remount
adb shell ps >> mem.txt
adb shell cat /proc/meminfo >> mem.txt
adb shell procrank >> mem.txt
adb shell dumpsys meminfo >> mem.txt
adb shell cat /proc/pagetypeinfo >> mem.txt
adb shell cat /proc/buddyinfo >> mem.txt
adb shell cat /proc/vmstat >> mem.txt
adb shell dumpsys procstats –-hours 2 >> mem.txt
pause
```

#### 采集

内存管控方案需要采集的信息有memfree、cache、buddyinfo等，用来解析内存状态；

CPU负载信息，用于内存决策时对系统状态的判断，来决策最终执行的策略；

进程内存状态，用于内存异常处理等的决策依据。

#### 识别

系统内存保障机制：识别系统空闲内存是否达到压力值去触发内存管控；

大内存应用保障：识别启动的应用是否为大内存应用，以及启动所需的内存大小；

系统内存异常识别：识别重要进程的内存是否超出阈值。

#### 决策

系统内存保障：根据系统空闲内存及CPU状态，决策执行reclaim操作还是kill操作，此时需要从应用管控内存；

大内存应用保障：决策是否启动快杀来给大内存应用提前释放内存；

系统内存异常：决策是否对内存异常的进程进行恢复。

#### 执行

Reclaim：通过iaware daemon下发到内核执行进程内存压缩操作；

Kill：通过进程清理服务清理进程内存。

## 10. 内存异常修复

主要包括两方面操作：内存碎片整理，对系统进程的内存占用异常进行监控处理。

使用JobScheduler来触发执行，只要满足系统idle、正在充电条件就会触发，触发后每隔N小时轮询一次。

内存碎片整理功能，基于kernel接口`/proc/sys/vm/compact_memory`实现。`CompactJobService->MemoryDefragger.compact->compactKernel`函数通过iawared向compact节点写入1，实现碎片整理。

CompactJobService在触发调用`defragger.compact`时，传入”background compact”，compact继而会调用LowMemoryFacade的reclaim功能，对关键进程的内存占用进行检测和修复。系统内存占用异常监控对以下进程占用内存超大进行修复。`system/bin/media_server`、`system_server`、`com.android.systemui`、`com.android.keyguard`等。

## 11. iaware绑核

iaware的CPU方案核心思路是保证前台程序的CPU资源，提高前台应用的用户感受，提升用户体验。iaware的CPU方案是将特定的任务加入到特定的分组，而在这些分组中的任务运行在特定的CPU上。

绑核设计的另一个重要设计思想是根据进程的负载情况，当负载达到一定的程度后将进程绑定到大核上运行。绑核是内核的一个基本特性，基于内核调度算法，通过改写内核任务的cpus_allowed来实现将特定的任务绑定到特定的CPU上调度执行，而内核的CPUSET功能是通过利用cgroup技术将绑核功能以分组的方式呈现给上层用户。

绑核是通过CPUSET分组实现的，通过将不同的分组对应到不同的CPU，而上层将特定的程序加入到特定的分组来绑定到特定的CPU上运行。

绑核策略原则是限制后台程序占用大核资源，保证前台程序的大核CPU资源。

**CPU set根组**：内核线程、native系统关键进程。刚创建的新进程，只短时驻留。
**Foreground组**：前台应用，即用户可见的应用，及可见应用依赖的应用和服务。
**Background组**：后台非关键应用，即用户无明显感知的不可见应用。
**system background**：系统后台组，即系统自带的后台次要进程。
**key background**：后台关键应用，即用户有明显感知的不可见应用。这类主要有：音乐播放、下载、运动传感（含GPS）、音频通话、录音等。另外根据用户习惯识别的TOP3 IM和TOP1 Email也可作为后台关键应用。新安装应用未经训练，可以先列入TOP IM或者TOP Email，这时可能会不只TOP3和TOP1。

## 12. 其他

[性能优化](http://hukai.me/android-performance-patterns/)
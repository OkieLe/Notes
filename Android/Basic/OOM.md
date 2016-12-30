## OOM

内存泄漏指由于疏忽或错误造成程序未能释放已经不再使用的内存。内存不是无穷无尽的，是有限的，如果申请了，使用了，用完没有释放，那么这块内存就一直无法被重新利用，最后再申请内存就找不到空闲的了（称为out of memory，OOM），可能导致程序逻辑错误崩溃。

内存泄漏可以根据发生的方式来分类：

- 常发性泄漏
  故障代码会被多次执行到，每次被执行的时候都会导致一块内存泄漏。
- 偶发性泄漏
  故障代码只有在特定场景或操作过程下才会被执行。常发性和偶发性是相对的。对于特定场景，偶发性的也许就变成了常发性的。所以测试场景和方法对检测内存泄漏至关重要。
- 一次性泄漏
  故障代码只会被执行一次或者由于逻辑的缺陷导致只有一块内存发生泄漏。比如，在类的构造函数中分配内存，在析构函数中却没有释放该内存。 
- 隐式泄漏
  程序在运行过程中不停的分配内存，但是直到结束的时候才释放内存。严格的说这里并没有发生内存泄漏，因为最终程序释放了所有申请的内存。但对于daemon或需要长时间运行的程序来讲，不及时释放可能最终耗尽所有内存。

另外不同的软件层都有可能存在内存泄漏：kernel层、native层和java层。不同的软件层泄漏的原理都差不多。

内存泄漏是较难检测的异常之一，除了常发性泄漏，其他都是难以检查到的。从用户使用程序的角度来看，内存泄漏本身不会产生什么危害，作为用户无法感觉到内存泄漏的存在，除非泄漏大量内存或一直累积直至消耗光内存，这时就有各种明显表现：

- 性能渐渐变差，因为要做各种内存回收工作。
- 直接出现逻辑错误崩溃，没有做好异常处理（error handling）。
- 本身没有问题，却引起其他程序异常。

### 现场信息

```
adb shell procrank 
```
- Unique set size (USS)： the set of pages that are unique to a process. This is the amount of memory that would be freed if the application gets terminated. USS is the best number to watch when initially suspicious of memory leaks in a process
- Proportional set size (PSS)：amount of memory shared with other processes, account in a way that the amount is divided evenly between the processes that share it. This is memory that would not be released if the process was terminated, but is indicative of the amount that this process is “contribution” to overall memory load.
- Resident set size (RSS)：the total memory actually held in RAM for a process. RSS can be misleading, because it reports the total allocations of the shared libraries that the process uses, even though a shared library is only loaded into memory once regardless of how many processes use it. RSS is not an accurate representation of the memory usage for a single process.
- Virtual set size (VSS)：the total accessible address space of a process. This size also includes memory that may not be resident in RAM like mallocsthat have been allocated but not written to. VSS is of very little use for determingreal memory usage of a process
```
adb shell dumpsys meminfo
```

## Native

### malloc

对于native内存泄漏，比较常见的一类是C堆内存泄漏，即调用malloc申请的内存没有及时释放造成的内存泄漏。

#### 调试方法

1. 打开malloc debug机制复现问题
   如何开启malloc debug请参考：MediaTek On-Line> Quick Start> 踩内存专题分析 之native踩内存调试方法
2. 检查是否存在内存泄漏。
   打开mtklogger，不断通过adb shell procrank -u > procrank.txt查看当前系统内存的使用情况。当看到被监控的进程占用内存(USS字段)超过了正常值很多，则可能存在内存泄漏。
3. 得到该进程内存使用分布情况($pid为被监控进程pid)
```
adb shell dumpsys meminfo $pid >meminfo_$pid.txt
adb shell procmem $pid > procmem_$pid.txt
adb shell kill -11 $pid>生成DB
```

#### 分析问题

复现问题后使用E-Consulter分析，分析完coredump会提供分析报告。如果存在超过128M以上的泄漏，那么分析报告会提示如下：

== C堆检查 ==
分配器: dlmalloc, 最多允许使用: 4GB, 最多使用: 188MB, 当前使用: 166MB, 泄露阈值: 128MB, 调试等级: 15

该堆已分配超过128MB (可能存在内存泄露), 以下列出分配最大尺寸和次数的调用栈:
大小: 1184字节, 已分配: 134604次
分配调用栈:
```
libc_malloc_debug_leak.so 0x40D5ECD4() + 16
libc_malloc_debug_leak.so leak_malloc() + 43
libc.so malloc() + 18
libsqlite.so sqlite3MemMalloc() + 14 <external/sqlite/dist/sqlite3.c:15298>
libsqlite.so mallocWithAlarm() + 86 <external/sqlite/dist/sqlite3.c:18862>
libsqlite.so sqlite3Malloc() + 16 <external/sqlite/dist/sqlite3.c:18895>
......
== 栈结束 ==
```
大小: 1184字节, 已分配: 5149次
分配调用栈:
```
libc_malloc_debug_leak.so 0x40D5ECD4() + 16
libc_malloc_debug_leak.so leak_malloc() + 43
libc.so malloc() + 18
libsqlite.so sqlite3MemMalloc() + 14 <external/sqlite/dist/sqlite3.c:15298>
libsqlite.so mallocWithAlarm() + 86 <external/sqlite/dist/sqlite3.c:18862>
......
== 栈结束 ==
```
大小: 1048584字节, 已分配: 1次
分配调用栈:
```
libc_malloc_debug_leak.so 0x40D5ECD4() + 16
libc_malloc_debug_leak.so leak_malloc() + 43
libc.so malloc() + 18
libsqlite.so sqlite3MemMalloc() + 14 <external/sqlite/dist/sqlite3.c:15298>
libsqlite.so mallocWithAlarm() + 86 <external/sqlite/dist/sqlite3.c:18862>
libsqlite.so sqlite3Malloc() + 16 <external/sqlite/dist/sqlite3.c:18895>
......
== 栈结束 ==
```
剩下就是检查代码逻辑，修复问题了

### mmap

native进程申请内存的方法很多，除了可以通过传统的堆分配函数malloc分配内存，还可以直接通过mmap映射内存。对于通过mmap分配的内存泄漏该如何调试呢？

#### 调试方法

1. 打开mmap debug机制复现
   在vendor/mediatek/proprietary/external/aee/config_external/init.aee.customer.rc添加:
```
on init
    export LD_PRELOAD libsigchain.so:libudf.so:...
```
"..."是原先LD_PRELOAD的内容，如果原先LD_PRELOAD没有内容，则":..."就不需要了。

重新打包bootimage并下载开机, 用adb输入:
```
adb shell setprop persist.debug.mmap.program app_process
adb shell setprop persist.debug.mmap.config 0x22002010
adb shell setprop persist.debug.mmap 1
adb reboot
```
注意：此类问题必须要抓到coredump，如何开启coredump请参考FAQ：[FAQ18022]L版本及之后的版本如何开启direct coredump? 

2. 检查是否存在内存泄漏。
   打开mtklogger，不断通过adb shell procrank -u > procrank.txt查看当前系统内存的使用情况。当看到被监控的进程占用内存(USS字段)超过了正常值很多，则可能存在内存泄漏。
3. 得到该进程内存使用分布情况($pid为被监控进程pid)
```
adb shell dumpsys meminfo $pid >meminfo_$pid.txt
adb shell procmem $pid > procmem_$pid.txt
adb shell kill -11 $pid>生成DB
```

#### 分析问题

复现问题后使用E-Consulter分析，分析完coredump会提供分析报告。如果存在超过200M以上的泄漏，那么分析报告会提示如下：

== mmap泄漏检查 ==
anon mmap: 818732KB

mmap已分配超过200MB (可能存在内存泄露), 以下列出分配最大尺寸和次数的调用栈:
大小: 1040384字节, 已分配: 505次
分配调用栈：
```
libudf.so mmap() + 138
libc.so pthread_create() + 586
libutils.so androidCreateRawThreadEtc() + 138
libutils.so androidCreateThreadEtc() + 14
libutils.so android::Thread::run() + 290
libstagefright_foundation.so android::ALooper::start() + 390
libmediandk.so 0x0000007F78EE4C24() + 162
libsoundpool.so android::Sample::doLoad() + 890
libsoundpool.so android::SoundPoolThread::doLoadSample() + 46
libsoundpool.so android::SoundPoolThread::run() + 78
== 栈结束 ==
```
大小: 104857600字节, 已分配: 1次
分配调用栈:
```
libudf.so mmap() + 138
libwebviewchromium_loader.so 0x0000007F72EA6FBC() + 38
libart.so 0x0000007F7DA33E10() + 150
...... 0x00000000063FFFFA()
== 栈结束 ==
```
大小: 1069056字节, 已分配: 61次
分配调用栈:
```
libudf.so mmap() + 138
libc.so pthread_create() + 586
libart.so art::Thread::CreateNativeThread() + 406
libart.so 0x0000007F7DA33E10() + 150
/dev/ashmem/dalvik-zygote space (deleted) 0x000000007226DFB2()
== 栈结束 ==
```
剩下就是检查代码逻辑，修复问题了。

## kernel内存回收

kernel的内存泄漏和native类似，不过原生的kernel就有集成内存回收和调试的功能。kernel设计的原则是不到万不得已就不做任何事情，比如内存尽量分配出去使用，实在没有内存才启动回收。

网络有些文章讲解内存回收机制，可以参考：[linux kernel内存回收机制](http://www.wowotech.net/memory_management/233.html)

### 内存回收机制

- 内存水位
  对于内存是否回收是针对每个zone考量的。而考量的参数有high、low和min，各个zone各一套，这3个参数的关系如下：
```
high > low > min
```
min以下的内存属于系统的保留内存，用以满足特殊使用，不会分配给用户态的申请。当内存可用量小于low时，kernel开始启动内核线程kswapd进行内存回收，当内存可用量降至min时，kernel直接进行直接回收（direct reclaim），即直接在应用程序的进程上下文中进行回收，再用回收上来的空闲页满足内存申请，因此会阻塞应用程序，带来一定的响应延迟，如果回收不到则会启动oom-killer通过杀程序回收内存，如果还是回收失败则会触发OOM。

kernel要么不回收内存，一旦开始回收内存，它将持续回收一直到可用的内存大于high。

- kswapd

kswapd是内存回收进程，且每个zone有一个，会定期监控和回收内存。

- OOM killer

实在要不到内存时，则启动oom killer，就是通过杀死比较肥的进程来回收内存。

如何评判要杀哪个进程呢？就是要给进程打分，根据分数排行，选择杀哪个进程，这里有篇文章讲解，请参考：[OOM相关的参数](http://www.wowotech.net/memory_management/oom.html)

- 隐含的规则：too small to fail

用GFP_KERNEL申请低阶（< 3） 连续内存（8 个连续页）分配的话，就不会返回NULL，如果不够内存，会通过回收机制回收，如果回收不到直接panic。

### 检测

OOM killer被启动时，会打印系统内存使用情况，有zone的情况，有swap的情况，等等。

log中各个栏位的含义：

active_anon、inactive_anon：native匿名内存。
active_file、inactive_file：native file cache，如果OOM一般会释放file cache，基本不用看这个栏位。
unevictable：无法回收的内存，包含了mlocked的数据。
mlocked：native通过mlock()函数将页面锁住，禁止被swap出去。
slab_reclaimable、slab_unreclaimable：kernel slub内存。
kernel_stack：内核栈内存，通过这个数据可以算出kernel中存在多少进程。
ARM64里进程数 = kernel_stack / 16K，ARM里进程数= kernel_stack / 8K

拿到这样的log，首先判断下是kernel泄漏还是native泄漏。看哪个栏位的值最大。最大的栏位就知道是native还是kernel泄漏了。如果是native泄漏，还可以看下面的进程的rss栏位，最多的那个进程很可能存在native内存泄漏了。

### kmemleak

kernel 2.6.31引入的工具，用于检查内存泄漏。

**原理**：创建一个进程每10分钟（默认）扫描系统内存然后打印泄漏的内存的信息。

**打开方法**

在config（位于arch/arm64/configs/xxx_defconfig或arch/arm/configs/xxx_defconfig）里设定：
```
CONFIG_DEBUG_KMEMLEAK=y
CONFIG_DEBUG_KMEMLEAK_EARLY_LOG_SIZE=1200
# CONFIG_DEBUG_KMEMLEAK_TEST is not set
# CONFIG_DEBUG_KMEMLEAK_DEFAULT_OFF is not set
```
或者用menuconfig配置：
```
Kernel Hacking> Kernel Memory Leak Detector
然后调大Maximum kmemleak early log entires
```
**检查**

编译下载后开机查看是否存在/sys/kernel/debug/kmemleak，如果不存在需要手动挂载
```
mount -t debugfs nodev /sys/kernel/debug/
```
查看内存泄漏信息
```
cat /sys/kernel/debug/kmemleak
```
得到的信息如下：
```
unreferenced object 0xf9061000 (size 512):
comm "insmod", pid 12750, jiffies 14401507 (age 110.217s)
hex dump (first 32 bytes):
1c 0f 00 00 01 12 00 00 2a 0f 00 00 01 12 00 00 ........*.......
38 0f 00 00 01 12 00 00 bc 0f 00 00 01 12 00 00 8...............
backtrace:
< c10b0001> create_object+0x114/0x1db
< c148b4d0> kmemleak_alloc+0x21/0x3f
< c10a43e9> __vmalloc_node+0x83/0x90
< c10a44b9> vmalloc+0x1c/0x1e
< f9055021> myfunc+0x21/0x23 hello_kernel
< f9058012> 0xf9058012
< c1001226> do_one_initcall+0x71/0x113
< c1056c48> sys_init_module+0x1241/0x1430
< c100284c> sysenter_do_call+0x12/0x22
< ffffffff> 0xffffffff
```
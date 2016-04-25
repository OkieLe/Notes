/proc/sysrq-trigger这个文件是最近调试内存稳定性的时候接触的，完整的内容可以参考内核目录下Documentation/sysrq.txt。用该功能必须将内核中的CONFIG_MAGIC_SYSRQ配置选项打开，但一般的发行版本都将此选项关闭了，该功能主要是用于调试的，想体验该功能的重新配置下内核。

下面介绍用法：

`echo b > /proc/sysrq-trigger`
立即重启机器，而且不会将缓冲区同步到硬盘，也不会卸载已挂载的硬盘
`echo c > /proc/sysrq-trigger`
使系统崩溃，如果配置了crashdump，崩溃后会生成dump文件
`echo d > /proc/sysrq-trigger`
列出系统中所有被持有的锁
`echo e > /proc/sysrq-trigger`
向系统中除init外的所有进程发出SIGTERM信号
`echo f > /proc/sysrq-trigger`
调用oom_kill杀死内存的hog进程
`echo g > /proc/sysrq-trigger`
kgdb会使用该项
`echo h > /proc/sysrq-trigger`
显示帮助信息
`echo i > /proc/sysrq-trigger`
向系统中除init外的所有进程发出SIGKILL信号
`echo j > /proc/sysrq-trigger`
Forcibly "Just thaw it" - filesystems frozen by the FIFREEZE ioctl（不明白）
`echo k > /proc/sysrq-trigger`
Secure Access Key (SAK) Kills all programs on the current virtual console. NOTE: See important comments below in SAK section.
`echo l > /proc/sysrq-trigger`
显示现在所有活动cpu的堆栈
`echo m > /proc/sysrq-trigger`
将当前内存信息dump到终端
`echo n > /proc/sysrq-trigger`
用来使实时任务可以设置nice值
`echo o > /proc/sysrq-trigger`
关闭系统
`echo p > /proc/sysrq-trigger`
将寄存器和flags dump到终端
`echo q > /proc/sysrq-trigger`
Will dump per CPU lists of all armed hrtimers (but NOT regular timer_list timers) and detailed information about all clockevent devices
`echo r > /proc/sysrq-trigger`
Turns off keyboard raw mode and sets it to XLATE。
`echo s > /proc/sysrq-trigger`
将尝试同步所有已挂载的文件系统
`echo u > /proc/sysrq-trigger`
将当前任务的列表和他们信息输出到终端
`echo v > /proc/sysrq-trigger`
强制恢复framebuffer console
`echo w > /proc/sysrq-trigger`
将进入uninterrupted状态的任务信息dump出来
`echo x > /proc/sysrq-trigger`
Used by xmon interface on ppc/powerpc platforms
`echo y > /proc/sysrq-trigger`
Show global CPU Registers [SPARC-64 specific]
`echo z > /proc/sysrq-trigger`
Dump the ftrace buffer
`echo '0'-'9' > /proc/sysrq-trigger`
Sets the console log level, controlling which kernel messages will be printed to your console. ('0', for example would make it so that only emergency messages like PANICs or OOPSes would make it to your console.)

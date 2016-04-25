在Android中使用启动脚本init.rc,可以在系统的初始化过程中进行一些简单的初始化操作。这个脚本被直接安装到目标系统的根文件系统中，被init可执行程序解析。

init.rc是在init启动后被执行的启动脚本

android启动文件系统后调用的第一个应用程序是/init，此文件的很重要的内容是解析了init.rc和init.xxx.rc两个配置文件，然后执行解析出来的任务。相关代码在android源代码/system/core/init/init.c文件中，如下：
```C++
parse_config_file("/init.rc");

/* pull the kernel commandline and ramdisk properties file in */
qemu_init();
import_kernel_cmdline(0);

get_hardware_name();
snprintf(tmp, sizeof(tmp), "/init.%s.rc", hardware);
parse_config_file(tmp);
```

从上面代码可以看到，第一个配置文件名称固定为init.rc,而第二个配置文件格式为init.xxx.rc，其中xxx部分的内容是从内核读取的，具体是读取文件/proc/cpuinfo中的Hardware部分，然后截取其部分内容。

从上面看init.xxx.rc中的xxx内容是取决平台的定义，例如：
```C++
parse_config_file(“init.qcom.rc”);
```

####配置文件的语法

#####配置文件的内容包含有4种：

```
动作(Action)
命令(Commands)
服务(Services)
选项(Options)
```

#####动作和命令一起使用，形式如下：

```init
on <trigger>
<command>
<command>
<command>
```
其中trigger是触发条件，也就是说在满足触发条件的情况下执行1个或多个相应的命令，举例如下：
```init
on property:persist.service.adb.enable=1
start adbd
```

#####服务和选项一起使用，形式如下：

```init
service <name> <pathname> [ <argument> ]*
<option>
<option>
...
```
上面内容解释为：
```
service 服务名称 服务对应的命令的路径 命令的参数
选项
选项
```
举例如下：
```init
service vold /system/bin/vold
socket vold stream 0660 root mount
service bootsound /system/bin/playmp3
user media
group audio
oneshot
```
>- vold和bootsound分别是两个服务的名称，/system/bin/vold和/system/bin/playmp3分别是他们所对应的可执行程序。
>- socket、user、group、oneshot就是配合服务使用的选项。其中oneshot选项表示该服务只启动一次，而如果没有oneshot选项，
>- 这个可执行程序会一直存在--如果可执行程序被杀死，则会重新启动。


#####选项是影响服务启动和运行的参数，主要的选项如下：

`disabled` 禁用服务，此服务开机时不会自动启动，但是可以在应用程序中手动启动它。

`socket <type> <name> <perm> [ <user> [ <group> ] ]`
>- 套接字 类型 名称 权限 用户 组
>- 创建一个名为/dev/socket/<name>，然后把它的fd传给启动程序
>- 类型type的值为dgram或者stream
>- perm表示该套接字的访问权限,user和group表示改套接字所属的用户和组，这两个参数默认都是0，因此可以不设置。

`user <username>`
执行服务前切换到用户<username>，此选项默认是root，因此可以不设置。

`group <groupname> [ <groupname> ]*`
执行服务前切换到组<groupname>,此选项默认是root,因此可以不设置

`capability [ <capability> ]+`
执行服务前设置linux capability，没什么用。

`oneshot`
服务只启动一次，一旦关闭就不能再启动。

`class <name>`
为服务指定一个类别，默认为"default"，同一类别的服务必须一起启动和停止

#####动作触发条件<trigger>

`boot` 首个触发条件，初始化开始(载入配置文件)的时候触发

`<name>=<value>`
当名为<name>的属性(property)的值为<value>的时候触发

`device-added-<path>`
路径为`<path>`的设置添加的时候触发

`device-removed-<path>`
路径为`<path>`的设置移除的时候触发

`service-exited-<name>`
名为`<name>`的服务关闭的时候触发

#####命令(Command)的形式

`exec <path> [ <argument> ]*`
复制(fork)和执行路径为`<path>`的应用程序，`<argument>`为该应用程序的参数，在该应用程序执行完前，此命令会屏蔽，

`export <name> <value>`
声明名为`<name>`的环境变量的值为<value>，声明的环境变量是系统环境变量，启动后一直有效。

`ifup <interface>`
启动名为<interface>的网络接口

`import <filename>`
加入新的位置文件，扩展当前的配置。

`hostname <name>`
设置主机名

`sysclktz<mins_west_of_gmt>`
设置系统时区(GMT为0)

`class_start <serviceclass>`
启动指定类别的所有服务

`class_stop <serviceclass>`
停止指定类别的所有服务

`domainname <name>`
设置域名

`insmod <path>`
加载路径为`<path>`的内核模块

`mkdir <path>`
创建路径为`<path>`目录

`mount <type> <device> <dir> [ <mountoption> ]*`
挂载类型为`<type>`的设备`<device>`到目录`<dir>`,`<mountoption>`为挂载参数，距离如下：
`mount ubifs ubi1_0 /data nosuid nodev`

`setkey`
暂时未定义

`setprop <name> <value>`
设置名为`<name>`的系统属性的值为`<value>`

`setrlimit <resource> <cur> <max>`
设置资源限制，

`start <service>`
启动服务(如果服务未运行)

`stop <service>`
停止服务(如果服务正在运行)

`symlink <target> <path>`
创建一个从`<path>`指向`<target>`的符号链接，举例：
`symlink /system/etc /etc`

`write <path> <string> [ <string> ]*`
打开路径为`<path>`的文件并将一个多这多个字符串写入到该文件中。

#####系统属性(Property)

android初始化过程中会修改一些属性，通过getprop命令我们可以看到属性值，这些属性指示了某些动作或者服务的状态，主要如下：

`init.action` 如果当前某个动作正在执行则init.action属性的值等于该动作的名称，否则为""

`init.command` 如果当前某个命令正在执行则init.command属性的值等于该命令的名称，否则为""

`init.svc.<name>` 此属性指示个名为`<name>`的服务的状态("stopped", "running", 或者 "restarting").

####init流程

init的源代码在文件：./system/core/init/init.c 中，init会一步步完成下面的任务：
1. 初始化log系统
2. 解析/init.rc和/init.%hardware%.rc文件  
3. 执行 early-init action in the two files parsed in step 2.  
4. 设备初始化，例如：在 /dev 下面创建所有设备节点，下载 firmwares.  
5. 初始化属性服务器，Actually the property system is working as a share memory.Logically it looks like a registry under Windows system.  
6. 执行 init action in the two files parsed in step 2.  
7. 开启 属性服务。
8. 执行 early-boot and boot actions in the two files parsed in step 2.  
9. 执行 Execute property action in the two files parsed in step 2.  
10. 进入一个无限循环 to wait for device/property set/child process exit events.例如,如果SD卡被插入，init会收到一个设备插入事件，它会为这个设备创建节点。系统中比较重要的进程都是由init来fork的，所 以如果他们他谁崩溃了，那么init 将会收到一个 SIGCHLD 信号，把这个信号转化为子进程退出事件， 所以在loop中，init 会操作进程退出事件并且执行*.rc 文件中定义的命令。

例如，在init.rc中，因为有：
```init
service zygote /system/bin/app_process -Xzygote /system/bin –zygote –start-system-server
    socket zygote stream 666
    onrestart write /sys/android_power/request_state wake
    onrestart write /sys/power/state on
```
所以，如果zygote因为启动某些服务导致异常退出后，init将会重新去启动它。
```C
int main(int argc, char **argv)
{
    ...
    //需要在后面的程序中看打印信息的话，需要屏蔽open_devnull_stdio()函数
    open_devnull_stdio();
    ...
    //初始化log系统
    log_init();
    //解析/init.rc和/init.%hardware%.rc文件
    parse_config_file(”/init.rc”);
    ...
    snprintf(tmp, sizeof(tmp), “/init.%s.rc”, hardware);
    parse_config_file(tmp);
    ...
    //执行 early-init action in the two files parsed in step 2.
    action_for_each_trigger(”early-init”, action_add_queue_tail);
    drain_action_queue();
    ...
    /* execute all the boot actions to get us started */
    /* 执行 init action in the two files parsed in step 2 */
    action_for_each_trigger(”init”, action_add_queue_tail);
    drain_action_queue();
    ...
    /* 执行 early-boot and boot actions in the two files parsed in step 2 */
    action_for_each_trigger(”early-boot”, action_add_queue_tail);
    action_for_each_trigger(”boot”, action_add_queue_tail);
    drain_action_queue();
    /* run all property triggers based on current state of the properties */
    queue_all_property_triggers();
    drain_action_queue();
    /* enable property triggers */  
    property_triggers_enabled = 1;   
    ...
    for(;;) {
        int nr, timeout = -1;
    ...
        drain_action_queue();
        restart_processes();
        if (process_needs_restart) {
            timeout = (process_needs_restart – gettime()) * 1000;
            ...
        }
    }
}
```

- 重要的数据结构两个列表，一个队列。

```C
static list_declare(service_list);
static list_declare(action_list);
static list_declare(action_queue);
```
>- \*.rc 脚本中所有 service关键字定义的服务将会添加到 service_list 列表中。
>- \*.rc 脚本中所有 on 关键开头的项将会被会添加到 action_list 列表中。
>- 每个action列表项都有一个列表，此列表用来保存该段落下的 Commands脚本解析过程：

```C
parse_config_file(”/init.rc”)
int parse_config_file(const char *fn)
{
    char *data;
    data = read_file(fn, 0);
    if (!data) return -1;
    parse_config(fn, data);
    DUMP();
    return 0;
}
static void parse_config(const char *fn, char *s)
｛
    …
    case T_NEWLINE:
        if (nargs) {
            int kw = lookup_keyword(args[0]);
            if (kw_is(kw, SECTION)) {
                state.parse_line(&state, 0, 0);
                parse_new_section(&state, kw, nargs, args);
            } else {
                state.parse_line(&state, nargs, args);
            }
            nargs = 0;
        }
   …
｝
```

parse_config会逐行对脚本进行解析，如果关键字类型为 SECTION ，那么将会执行 parse_new_section() 类型为 SECTION 的关键字有： on 和 sevice 关键字类型定义在 Parser.c (system/core/init) 文件中
Parser.c (system/core/init)
```C
#define SECTION 0×01
#define COMMAND 0×02
#define OPTION  0×04
关键字        属性      
capability,  OPTION,  0, 0)
class,       OPTION,  0, 0)
class_start, COMMAND, 1, do_class_start)
class_stop,  COMMAND, 1, do_class_stop)
console,     OPTION,  0, 0)
critical,    OPTION,  0, 0)
disabled,    OPTION,  0, 0)
domainname,  COMMAND, 1, do_domainname)
exec,        COMMAND, 1, do_exec)
export,      COMMAND, 2, do_export)
group,       OPTION,  0, 0)
hostname,    COMMAND, 1, do_hostname)
ifup,        COMMAND, 1, do_ifup)
insmod,      COMMAND, 1, do_insmod)
import,      COMMAND, 1, do_import)
keycodes,    OPTION,  0, 0)
mkdir,       COMMAND, 1, do_mkdir)
mount,       COMMAND, 3, do_mount)
on,          SECTION, 0, 0)
oneshot,     OPTION,  0, 0)
onrestart,   OPTION,  0, 0)
restart,     COMMAND, 1, do_restart)
service,     SECTION, 0, 0)
setenv,      OPTION,  2, 0)
setkey,      COMMAND, 0, do_setkey)
setprop,     COMMAND, 2, do_setprop)
setrlimit,   COMMAND, 3, do_setrlimit)
socket,      OPTION,  0, 0)
start,       COMMAND, 1, do_start)
stop,        COMMAND, 1, do_stop)
trigger,     COMMAND, 1, do_trigger)
symlink,     COMMAND, 1, do_symlink)
sysclktz,    COMMAND, 1, do_sysclktz)
user,        OPTION,  0, 0)
write,       COMMAND, 2, do_write)
chown,       COMMAND, 2, do_chown)
chmod,       COMMAND, 2, do_chmod)
loglevel,    COMMAND, 1, do_loglevel)
device,      COMMAND, 4, do_device)
```
parse_new_section()中再分别对 service 或者 on 关键字开头的内容进行解析。
```C
    …
    case K_service:
        state->context = parse_service(state, nargs, args);
        if (state->context) {
            state->parse_line = parse_line_service;
            return;
        }
        break;
    case K_on:
        state->context = parse_action(state, nargs, args);
        if (state->context) {
            state->parse_line = parse_line_action;
            return;
        }
        break;
    }
    …
```
对 on 关键字开头的内容进行解析
```C
static void *parse_action(struct parse_state *state, int nargs, char **args)
{
    …
    act = calloc(1, sizeof(*act));
    act->name = args[1];
    list_init(&act->commands);
    list_add_tail(&action_list, &act->alist);
    …
}
```
对 service 关键字开头的内容进行解析
```C
static void *parse_service(struct parse_state *state, int nargs, char **args)
{
    struct service *svc;
    if (nargs name = args[1];
    svc->classname = “default”;
    memcpy(svc->args, args + 2, sizeof(char*) * nargs);
    svc->args[nargs] = 0;
    svc->nargs = nargs;
    svc->onrestart.name = “onrestart”;
    list_init(&svc->onrestart.commands);
    //添加该服务到 service_list 列表
    list_add_tail(&service_list, &svc->slist);
    return svc;
}
```
服务的表现形式:
```init
service   [  ]*
…
```
申请一个service结构体,然后挂接到service_list链表上,name 为服务的名称 pathname 为执行的命令 argument 为命令的参数。之后的 option 用来控制这个service结构体的属性,parse_line_service 会对 service关键字后的 内容进行解析并填充到 service 结构中 ，当遇到下一个service或者on关键字的时候此service选项解析结束。
例如：
```init
service zygote /system/bin/app_process -Xzygote /system/bin –zygote –start-system-server
    socket zygote stream 666
    onrestart write /sys/android_power/request_state wake
```

- 服务名称为：                           zygote
- 启动该服务执行的命令：                 /system/bin/app_process
- 命令的参数：                           -Xzygote /system/bin –zygote –start-system-server
- socket zygote stream 666： 创建一个名为：/dev/socket/zygote 的 socket ，类型为：stream

当*.rc 文件解析完成以后：
action_list 列表项目如下：
```init
on init
on boot
on property:ro.kernel.qemu=1
on property:persist.service.adb.enable=1
on property:persist.service.adb.enable=0
init.marvell.rc 文件
on early-init
on init
on early-boot
on boot
service_list 列表中的项有：
service console
service adbd
service servicemanager
service mountd
service debuggerd
service ril-daemon
service zygote
service media
service bootsound
service dbus
service hcid
service hfag
service hsag
service installd
service flash_recovery
```
状态服务器相关：

在init.c 的main函数中启动状态服务器。
`property_set_fd = start_property_service();`

状态读取函数：
```C
Property_service.c (system/core/init)
const char* property_get(const char *name)
Properties.c (system/core/libcutils)
int property_get(const char *key, char *value, const char *default_value)
```
状态设置函数：
```C
Property_service.c (system/core/init)
int property_set(const char *name, const char *value)
Properties.c (system/core/libcutils)
int property_set(const char *key, const char *value)
```

在终端模式下我们可以通过执行命令 setprop  
setprop 工具源代码所在文件： Setprop.c (system/core/toolbox)
Getprop.c (system/core/toolbox):        property_get(argv[1], value, default_value);
Property_service.c (system/core/init)中定义的状态读取和设置函数仅供init进程调用，
```
handle_property_set_fd(property_set_fd);
  property_set()   //Property_service.c (system/core/init)
    property_changed(name, value) //Init.c (system/core/init)
      queue_property_triggers(name, value)
      drain_action_queue()
```

只要属性一改变就会被触发，然后执行相应的命令：  
例如：
在init.rc 文件中有
```
on property:persist.service.adb.enable=1
  start adbd
on property:persist.service.adb.enable=0
  stop adbd
```
所以如果在终端下输入：
```shell
setprop property:persist.service.adb.enable 1或者0
```
那么将会开启或者关闭adbd 程序。

执行action_list 中的命令：
从action_list 中取出 act->name 为 early-init 的列表项，再调用 action_add_queue_tail(act)将其插入到 队列 action_queue 尾部。drain_action_queue() 从action_list队列中取出队列项 ，然后执行act->commands

列表中的所有命令。
所以从  ./system/core/init/init.c main()函数的程序片段：
```C
action_for_each_trigger(”early-init”, action_add_queue_tail);
drain_action_queue();
action_for_each_trigger(”init”, action_add_queue_tail);
drain_action_queue();
action_for_each_trigger(”early-boot”, action_add_queue_tail);
action_for_each_trigger(”boot”, action_add_queue_tail);
drain_action_queue();
/* run all property triggers based on current state of the properties */
queue_all_property_triggers();
drain_action_queue();
```
可以看出，在解析完init.rc init.marvell.rc 文件后，action 命令执行顺序为：

- 执行act->name 为 early-init，act->commands列表中的所有命令
- 执行act->name 为 init，            act->commands列表中的所有命令
- 执行act->name 为 early-boot，act->commands列表中的所有命令
- 执行act->name 为 boot，            act->commands列表中的所有命令

关键的几个命令：
- class_start default   启动所有service 关键字定义的服务。
- class_start 在act->name为boot的 act->commands列表中，所以当 class_start 被触发后，实际上调用的是函数 do_class_start（）
```C
int do_class_start(int nargs, char **args)
{
        /* Starting a class does not start services
         * which are explicitly disabled.  They must
         * be started individually.
         */
    service_for_each_class(args[1], service_start_if_not_disabled);
    return 0;
}
void service_for_each_class(const char *classname,
                            void (*func)(struct service *svc))
{
    struct listnode *node;
    struct service *svc;
    list_for_each(node, &service_list) {
        svc = node_to_item(node, struct service, slist);
        if (!strcmp(svc->classname, classname)) {
            func(svc);
        }
    }
}
```
因为在调用 parse_service() 添加服务列表的时候，所有服务 svc->classname 默认取值：”default”，
所以 service_list 中的所有服务将会被执行。

来源： <http://blog.csdn.net/zhenwenxian/article/details/7506392>

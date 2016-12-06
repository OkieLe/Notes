# adb

### 名词

ADB：Android Debug Bridge
ADT：Android Development Tools

## adb工具

ADB是Android为开发人员提供的调试工具([官方文档](https://developer.android.com/studio/command-line/adb.html))，方便从电脑侧对手机进行一些操作，或者获取手机的一些状态信息。ADB是一个CS架构的工具，包含三部分：
1. 客户端：运行在开发用的PC设备上，用来发送命令，在命令行执行adb命令就能启动
2. Daemon(adbd)：运行在开发用的移动设备(手机)上，用来执行命令，是作为后台进程运行的
3. 服务端：运行在开发用的PC设备上，用来管理客户端和Daemon的通信，是作为后台进程运行的

adbd可以认为Daemon，是放在手机中的，响应PC侧的adb命令。adb是PC侧的可执行程序，一般随Google官方SDK一起发布，里面包含了服务端和客户端。

### adb连接

当启动一个adb客户端时(执行adb命令)，客户端先检查是否有adb服务进程正在运行，如果没有则启动服务进程。当服务起来，就会绑定TCP端口5037监听所有客户端发送的命令(所有客户端都是用5037与服务进行通信)。

然后服务通过扫描5555-5585间奇数端口，定位并建立与所有正在运行的虚拟机和设备对象的连接。当服务端发现一个adb daemon，就建议与那个端口的连接。每个设备用连续的两个端口号：偶数端口给console连接，奇数给adb链接。比如：
> Emulator 1, console: 5554
> Emulator 1, adb: 5555
> Emulator 2, console: 5556
> Emulator 2, adb: 5557
> ...

建立连接后，就可以用adb命令访问这些虚拟机或者设备。

### 打开adb连接

打开设置中“开发选项”下“USB调试”功能(可能被厂商修改过)，用USB连接手机到电脑即可。然后就可以执行adb命令`adb devices`查看手机是否已连接，adb可执行程序在`android_sdk/platform-tools/`中。

Android4.2.2开始，连接到电脑后，手机中会弹出授权确认框要求用户进行授权，授权后才可以使用完整的adb命令。

另外也可以通过WIFI连接adb(替代USB连接)：
1. 连接开发电脑和手机到同一个WIFI热点
2. 通过USB连接手机到电脑
3. 让手机监听端口5555`adb tcpip 5555`
4. 断开USB连接
5. 从“设置 > 关于手机 >状态 > IP地址”中查看IP地址
6. 连接该手机`adb connect device_ip_address`
7. 确认：
```shell
$ adb devices
List of devices attached
device_ip_address:5555 device
```

> 如果连接断开，只要WIFI没有变化，可以再执行`adb connect`重新连接
> 如果不行，可以通过`adb kill-server`重置服务，从头再来

### 常用命令

- `adb devices`
  列出已连接设备，其中可以看到设备序列号和设备状态。

- `adb -s serial_number command`
  向特定设备发送命令

- `adb forward tcp:6100 tcp:7100`
  TCP端口重定向

- `adb pull remote local/adb push local remote`
  文件复制

- `adb shell [shell_command]`
  无附加命令时启动一个Linux shell，可以执行手机支持的各种shell命令(受Linux用户权限限制)，或者直接附加手机支持的shell命令，直接执行。

- `adb shell screencap /sdcard/screen.png`
  截图

- `adb shell screenrecord /sdcard/demo.mp4`
  录制屏幕

*下面看adb的实现，相关的代码在ADB_ROOT(`system/core/adb`)中。*

## adbd

adbd定义在`system/core/rootdir/init.usb.rc`中，adbd的控制是通过`init.<platform>.usb.rc`中的属性触发器，如下面代码第二部分。
```
# adbd is controlled via property triggers in init.<platform>.usb.rc
service adbd /sbin/adbd --root_seclabel=u:r:su:s0
    class core
    socket adbd stream 660 system system
    disabled
    seclabel u:r:adbd:s0

# trigger to control adbd
on property:sys.usb.config=adb
    write /sys/class/android_usb/android0/enable 0
    write /sys/class/android_usb/android0/idVendor 18d1
    write /sys/class/android_usb/android0/idProduct 4EE7
    write /sys/class/android_usb/android0/functions ${sys.usb.config}
    write /sys/class/android_usb/android0/enable 1
    start adbd
    setprop sys.usb.state ${sys.usb.config}
```

adbd的入口方法在`${ADB_ROOT}/daemon/main.cpp`中。

1. adbd_main(); //真正的启动流程
2. init_transport_registration(); //注册Socket
3. adbd_auth_init(); //授权信息初始化
4. drop_privileges(); //确定启动的用户、组等，adb是否root在此确定
5. install_listener(); //注册本地监听器
6. usb_init();->adb_thread_create(); //监听USB设备
7. local_init();->adb_thread_create(); //监听虚拟器
8. init_jdwp(); //初始化Socket设置
9. fdevent_loop(); //死循环等待客户端请求

命令处理：
```C++
void fdevent_loop()
{
    set_main_thread();
#if !ADB_HOST
    fdevent_subproc_setup();
#endif // !ADB_HOST

    while (true) {
        if (terminate_loop) {
            return;
        }

        D("--- --- waiting for events");
        fdevent_process();
        while (!g_pending_list.empty()) {
            fdevent* fde = g_pending_list.front();
            g_pending_list.pop_front();
            fdevent_call_fdfunc(fde);
        }
    }
}
```
`fdevent_process()`读取传入命令，并存入`g_pending_list`，然后逐个调用`fdevent_call_fdfunc()`进行处理。

## adb

adb的入口方法在`${ADB_ROOT}/client/main.cpp`中。
```
adb_commandline(); //Only method to deal with User input.
```
adb客户端是实时响应的，每次用户输入一个adb命令并回车后，这个main被调起，然后`adb_commandline()`解析命令以及附加的参数，并将命令发送到服务端。

- 需要重点关注的方法是`adb_connect()`，这是用来处理服务的启动和连接，必要时通过`launch_server()`启动服务进程。
- 命令的发送和结果的读取是通过`adb_io.h`中的方法进行的，本质上是Socket连接的方式。

### adb server

adb server是通过`launch_server()`启动的，方法中主要的工作是执行`adb -L %s fork-server server --reply-fd %d`，放在新建的一个子进程中。

这时再次进入`adb_commandline()`，条件变化后进入`adb_server_main()`，流程类似`adbd_main()`，初始化USB连接、Socket、授权信息后，进入死循环等待请求：
```C++
int adb_server_main(int is_daemon, const std::string& socket_spec, int ack_reply_fd) {
    ...
    init_transport_registration();

    usb_init();
    local_init(DEFAULT_ADB_LOCAL_TRANSPORT_PORT);
    if (install_listener(socket_spec, "*smartsocket*", nullptr, 0, nullptr, &error)) {
        fatal("could not install *smartsocket* listener: %s", error.c_str());
    }
    adb_auth_init();

    ...
    D("Event loop starting");
    fdevent_loop();

    return 0;
}
```

adbd和adb server的主要区别在`usb_init()`中，针对不同的目标平台实现不同，需要的可自行研究代码。

## 通信

adb客户端和adb server、adb server和adbd本质上都是通过Socket通信的，但是具体到目前的实现，关键的类是`atransport`。adbd和adb server启动时都会通过`init_transport_registration()`创建一个Socket pair处理异步的注册事件，用于发送和接收数据。
```
void init_transport_registration(void)
{
    int s[2];

    if(adb_socketpair(s)){
        fatal_errno("cannot open transport registration socketpair");
    }
    D("socketpair: (%d,%d)", s[0], s[1]);

    transport_registration_send = s[0];
    transport_registration_recv = s[1];

    fdevent_install(&transport_registration_fde,
                    transport_registration_recv,
                    transport_registration_func,
                    0);

    fdevent_set(&transport_registration_fde, FDE_READ);
}
```

### 附加说明

上面调用的方法`fdevent_install()`是一个事件注册方法，即`transport_registration_recv`上的事件由`transport_registration_func`响应。

`transport_registration_recv`中通过同样的方法注册了`transport_socket_events`进行监听Socket端口的消息。然后又创建两个线程：`write_transport_thread`和 `read_transport_thread`，分别进入死循环，用于发送和接收数据。

两个线程中各自用`write_to_remote()`和`read_from_remote()`发送和接收消息，对应到`transport_usb.cpp`(USB设备)和`transport_local.cpp`(模拟器)中的`remote_write()`和`remote_read()`。

`install_listener()`中注册的监听器`ss_listener_event_func`->`local_socket_event_func`会将本地收到的Packet转发，最终处理消息的方法是`handle_packet()`(`adb.cpp`)。
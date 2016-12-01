# Android Recovery

Android利用Recovery模式，进行恢复出厂设置，OTA升级，patch升级及firmware升级。recovery.img是目录`bootable/recovery`下的源代码编译生成。Recovery Image 的生成规则在文件`build/core/Makefile`中定义。

## 重启-关机流程

1. `PowerManager.reboot(reason); // reason = "recovery" or "recovery-update"`
2. `PowerManagerService.lowLevelReboot(reason);`
3. `SystemProperties.set("sys.powerctl", "reboot,recovery");`
4. `android_reboot(); //system/core/libcutils/android_reboot.c`
5. `syscall(__NR_reboot, LINUX_REBOOT_MAGIC1, LINUX_REBOOT_MAGIC2, LINUX_REBOOT_CMD_RESTART2, arg);`
6. `kernel_restart(); -> machine_restart();// kernel/x-x/kernal/reboot.c`

## bootloader

bootloader 就是在操作系统内核运行之前运行的一段小程序。通过这段小程序，可以初始化硬件设备、建立内存空间的映射图，从而将系统的软硬件环境带到一个合适的状态，以便为最终调用操作系统内核准备好正确的环境。

1. sbl1的功能是对硬件进行初始化并加载其他模块，需要加载的模块信息按顺序保存在sbl1中，对应每个模块的数据是一段大小为0x64字节的模块信息数据内，sbl1中有一个循环负责验证和加载所有需要的其他模块（tz，rpm，appsbl等），加载代码会根据模块信息内的数据调用不同的加载器加载和验证的代码。
2. emmc_appsboot.mbn就是appsbl模块，模块格式为bin，文件最前面的0x28字节的头部描述了bin的加载地址等信息，后面的数据就是实际加载到内存中的映像，整个bootloader中这个模块的代码量最大（很大一部分是openssl的代码），linux内核的验证和加载（正常启动和Recovery模式）等代码都包含在这个模块内。目前高通平台代码，`bootable/bootloader/lk`目录中是bootloader相关的开源代码。

> 整体代码量大，细节还没有看明白，暂时只看与recovery相关的。

### 启动模式选择

从appsbl/aboot启动到正常模式，还是recovery、fastboot等模式，进入recovery模式关键在于设置`boot_into_recovery`为1，然后调用`emmc_recovery_init()`。

appsbl/aboot根据之前用户的操作，设置好环境变量以及相关信息，启动bootimage或者recoveryimage。

```C
// bootable/bootloader/lk/app/aboot/aboot.c
void aboot_init(const struct app_descriptor *app)
{
    unsigned reboot_mode = 0;
    ...
    read_device_info(&device);
    read_allow_oem_unlock(&device);

    /* Display splash screen if enabled */
#if DISPLAY_SPLASH_SCREEN
    if (!check_alarm_boot()) {
#if ENABLE_WBC
        /* Wait if the display shutdown is in progress */
        while(pm_app_display_shutdown_in_prgs());
        if (!pm_appsbl_display_init_done())
            target_display_init(device.display_panel);
        else
            display_image_on_screen();
#else
        target_display_init(device.display_panel);
#endif
        dprintf(SPEW, "Display Init: Done\n");
    }
#endif

    target_serialno((unsigned char *) sn_buf);
    dprintf(SPEW,"serial number: %s\n",sn_buf);

    memset(display_panel_buf, '\0', MAX_PANEL_BUF_SIZE);

    /*
     * Check power off reason if user force reset,
     * if yes phone will do normal boot.
     */
    if (is_user_force_reset())
        goto normal_boot;

    /* Check if we should do something other than booting up */
    if (keys_get_state(KEY_VOLUMEUP) && keys_get_state(KEY_VOLUMEDOWN))
    {
        reboot_device(EMERGENCY_DLOAD);
        boot_into_fastboot = true;
    }
    if (!boot_into_fastboot)
    {
        if (keys_get_state(KEY_HOME) || keys_get_state(KEY_VOLUMEUP))
            boot_into_recovery = 1;
        if (!boot_into_recovery &&
            (keys_get_state(KEY_BACK) || keys_get_state(KEY_VOLUMEDOWN)))
            boot_into_fastboot = true;
    }
    #if NO_KEYPAD_DRIVER
    if (fastboot_trigger())
        boot_into_fastboot = true;
    #endif

#if USE_PON_REBOOT_REG
    reboot_mode = check_hard_reboot_mode();
#else
    reboot_mode = check_reboot_mode();
#endif
    if (reboot_mode == RECOVERY_MODE)
    {
        boot_into_recovery = 1;
    }
    else if(reboot_mode == FASTBOOT_MODE)
    {
        boot_into_fastboot = true;
    }
    else if(reboot_mode == ALARM_BOOT)
    {
        boot_reason_alarm = true;
    }
    ...

normal_boot:
    if (!boot_into_fastboot)
    {
        if (target_is_emmc_boot())
        {
            if(emmc_recovery_init())
                dprintf(ALWAYS,"error in emmc_recovery_init\n");
            ...
            boot_linux_from_mmc();
        }
        else
        {
            recovery_init();
            boot_linux_from_flash();
        }
        dprintf(CRITICAL, "ERROR: Could not do normal boot. Reverting "
            "to fastboot mode.\n");
    }

    /* We are here means regular boot did not happen. Start fastboot. */

    /* register aboot specific fastboot commands */
    aboot_fastboot_register_commands();

    /* dump partition table for debug info */
    partition_dump();

    /* initialize and start fastboot */
    fastboot_init(target_get_scratch_address(), target_get_max_flash_size());
#if FBCON_DISPLAY_MSG
    display_fastboot_menu();
#endif
}
```

recovery.img 和 boot.img 的区别不大，主要是 init 脚本不一样，recovery 的 init 脚本(`bootable/recovery/etc/init.rc`)相对简单，系统起来后只运行 ueventd、recovery、adbd 三个服务。

## Recovery模式

init.rc中定义了recovery启动，也就是recovery.img被当做系统分区启动时，在init的过程中启动了recovery。Recovery模式的界面是在`bootable/recovery/screen_ui.*`中定义，定制也可以在此修改。

有时候 Android 需要不同的模式互相协助来完成一项任务，这样不同模式之间就要有一种机制来交换信息。

- Recovery和 Bootloader 之间是通过 misc 分区来传递信息的，aboot中获取的信息通过结构体`recovery_message(bootable/bootloader/lk/app/aboot/recovery.h)`封装，写入misc分区。`emmc_recovery_init()`中主要工作就是解析命令后存入misc分区。recovery运行时，从对应的misc读取后，保存在`bootloader_message(bootable/recovery/bootloader.h)`结构体中。

- 另外Recovery 和 Android 之间是通过 cache 分区下的recovery来传递信息的：
> /cache/recovery/command：输入，工具的命令行，每行一个参数
> /cache/recovery/log：输出，recovery运行的log记录
> recovery.command文件中支持的命令如下：
> - update_package=path：验证并安装OTA包
> - wipe_data：擦除userdata和cache，重启
> - wipe_cache：擦除cache，重启
> - set_encrypted_filesystem=on|off：加密、解密文件系统
> - just_exit：退出并重启
>   操作完成后，`/cache/recovery/command`被擦除。

### 擦除数据/Factory Reset

**BCB**：bootloader control block，即上一部分说的misc分区。

用户恢复出厂设置流程如下：
1. 用户选择设置中“恢复出厂设置”
2. 主系统写入`--wipe_data`到`/cache/recovery/command`
3. 重启进入recovery模式
4. get_args()向BCB写入`boot-recovery`和`--wipe_data`
5. erase_volume("/data")
6. erase_volume("/cache")
7. finish_recovery()擦除BCB
8. 重启手机到正常模式

### OTA升级

1. 下载OTA包到`/cache/some-filename.zip`
2. 写入command：`--update_package=/cache/some-filename.zip`
3. 重启到recovery模式
4. get_args()向BCB写入`boot-recovery`和`--update_package=...`
5. install_package()开始尝试安装
6. finish_recovery()擦除BCB
7. **如果安装失败**
   7a. prompt_and_wait()显示提示，等用户处理
   7b. 用户重启(拔电池等方式)
8. 重启手机到正常模式

> 想了解更多细节，可以自行阅读bootable中的代码。
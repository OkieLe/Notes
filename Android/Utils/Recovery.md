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

init.rc中定义了recovery启动，也就是recovery.img被当做系统分区启动时，在init的过程中启动了recovery。

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

### 厂商定制

Recovery模式的界面是在`bootable/recovery/screen_ui.*`中定义，定制也可以在此修改。([官方说明](https://source.android.com/devices/tech/ota/device_code.html))。

厂商可以在BoardConfig.mk中定义`TARGET_RECOVERY_FSTAB`替换recovery下的fstab。

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

### 其他

**sideloading**：当手机无法正常启动去下载OTA包的时候，可以用这个功能进行升级。一般可以用SD卡升级替代，但是对于不支持扩展SD卡的手机，这个是一种完善的机制。这个支持两种方式，从cache分区或者通过adb。

adb的方式命令如下：
```
adb sideload filename
```

**A/B系统升级**：大致逻辑是采用两套分区，升级时使用当前未使用的分区，当升级完成后再重启进入升级后的分区。如果升级失败则还进入老的分区，检测分区是否异常使用dm-verity。升级使用一个后台服务update_engine，支持这个功能还需要修改bootloader、分区表、编译程序和OTA包。([官方说明](https://source.android.com/devices/tech/ota/ab_updates.html))

## 升级包

Block-Based OTA：基于块的OTA，Android L加入，保证每个分区升级后的内容完全一致，每个分区一个文件，只生成一个patch。L之前的版本使用文件对比的方式，只保证每个文件内容、权限和模式一样，时间戳等元数据可以不同。这也是支持dm-verity的前提，升级后分区内容和用fastboot刷入的一样。

### 制作升级包

`build/tools/releasetools`目录中的工具`ota_from_target_files`可以用来制作OTA升级包。

升级包制作要用到编译版本生成的target-files.zip文件，编译方法：
```shell
# first, build the target-files .zip
$ . build/envsetup.sh && lunch tardis-eng
$ mkdir dist_output
$ make dist DIST_DIR=dist_output
```

1. 制作完整包：
```
$ ./build/tools/releasetools/ota_from_target_files dist_output/tardis-target_files.zip ota_update.zip
```
2. 制作差分包：
```
$ ./build/tools/releasetools/ota_from_target_files -i PREVIOUS-tardis-target_files.zip dist_output/tardis-target_files.zip incremental_ota_update.zip
```
3. 制作Block-based OTA包需要添加` --block `参数

### 升级包内容

升级包中有一个可执行程序`META-INF/com/google/android/update-binary`，就是安装升级包的程序。还有一个脚本`META-INF/com/google/android/update-script`，脚本使用edify进行解释，edify源码放在`bootable/recovery/edify`。

系统默认的升级程序代码放在`bootable/recovery/updater`中。这个程序和脚本可以用其他自定义的替换掉。一般厂商会在`/device/*/*`目录下进行定制。

### 安装升级包

不论是通过OTA、SD卡，还是直接adb加载升级包，只是升级包所在的位置不同，最终安装升级包的逻辑是一样的。安装升级包代码在`install_package();//bootable/recovery/install.cpp`

1. `setup_install_mounts()`检查分区挂载情况，确保tmp和cache两个分区已挂载
2. `ensure_path_mounted()`确保安装包所在分区已挂载
3. `verify_file()`验证安装包签名有效性
4. `mzOpenZipArchive()/OpenArchiveFromMemory()`打开压缩包，读出内容
5. `try_update_binary()`执行升级包中的脚本(先解压到/tmp)，按照脚本进行升级（平台实现不同）

> 想了解更多细节，可以自行阅读bootable中的代码。
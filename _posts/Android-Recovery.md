---
published: true
title: Android Recovery模式
date: {}
Tags:
  - Android
  - Recovery mode
---

## Recovery简介

Android利用Recovery模式，进行恢复出厂设置，OTA升级，patch升级及firmware升级。升级一般通过运行升级包中的META-INF/com/google/android/update-script脚本来执行自定义升级，脚本中是一组recovery系统能识别的UI控制，文件系统操作命令，例如write_raw_image（写FLASH分区），copy_dir（复制目录）。该包一般被下载至SDCARD和CACHE分区下。如果对该包内容感兴趣，可以从http://forum.xda-developers.com/showthread.php?t=442480 下载JF升级包来看看。升级中还涉及到包的数字签名，签名方式和普通JAR文件签名差不错。公钥会被硬编译入recovery，编译时生成在：out/target/product/XX/obj/PACKAGING/ota_keys_inc_intermediates/keys.inc

### G1中的三种启动模式

MAGIC KEY:
1. camera + power：bootloader模式，ADP里则可以使用fastboot模式
2. home + power：recovery模式
3. 正常启动, Bootloader正常启动，又有三种方式，按照BCB（Bootloader Control Block, 下节介绍）中的command分类：
 - command == 'boot-recovery' → 启动recovery.img。recovery模式
 - command == 'update-radio/hboot' → 更新firmware（bootloader)
4. 其他 → 启动boot.img

### Recovery涉及到的其他系统及文件

**CACHE分区文件**

Recovery 工具通过NAND cache分区上的三个文件和主系统打交道。主系统（包括恢复出厂设置和OTA升级）可以写入recovery所需的命令，读出recovery过程中的LOG和intent。

- /cache/recovery/command： recovery命令，由主系统写入。所有命令如下：
``` bash
--send_intent=anystring - write the text out to recovery.intent
--update_package=root:path - verify install an OTA package file
--wipe_data - erase user data (and cache), then reboot
--wipe_cache - wipe cache (but not user data), then reboot
```
- /cache/recovery/log：recovery过程日志，由主系统读出
- /cache/recovery/intent：recovery输出的intent

**MISC分区内容**

Bootloader Control Block (BCB) 存放recovery bootloader message。结构如下：
``` C
struct bootloader_message {
	char command[32];
	char status[32]; // 未知用途
	char recovery[1024];
};
```

command可以有以下两个值
1. “boot-recovery”：标示recovery正在进行，或指示bootloader应该进入recovery mode
2. “update-hboot/radio”：指示bootloader更新firmware

recovery内容
“recovery\n
\n
”
其中recovery command为CACHE:/recovery/command命令

### 两种Recovery Case

**FACTORY RESET（恢复出厂设置）**
1. 用户选择“恢复出厂设置”
2. 设置系统将"--wipe_data"命令写入/cache/recovery/command
3. 系统重启，并进入recover模式（/sbin/recovery）
4. get_args() 将 "boot-recovery"和"--wipe_data"写入BCB
5. erase_root() 格式化（擦除）DATA分区
6. erase_root() 格式化（擦除）CACHE分区
7. finish_recovery() 擦除BCB
8. 重启系统

**OTA INSTALL（OTA升级）**
1. 升级系统下载 OTA包到/cache/some-filename.zip
2. 升级系统写入recovery命令"--update_package=CACHE:some-filename.zip"
3. 重启，并进入recovery模式
4. get_args() 将"boot-recovery" 和 "--update_package=..." 写入BCB
5. install_package() 作升级
6. finish_recovery() 擦除 BCB
7. 如果安装包失败 prompt_and_wait() 等待用户操作，选择ALT+S或ALT+W 升级或恢复出厂设置
8. main() 调用 maybe_install_firmware_update()
 1) 如果包里有hboot/radio的firmware则继续，否则返回
 2) 将 "boot-recovery" 和 "--wipe_cache" 写入BCB
 3) 将 firmware image写入cache分区
 4) 将 "update-radio/hboot" 和 "--wipe_cache" 写入BCB
 5) 重启系统
 6) bootloader自身更新firmware
 7) bootloader 将 "boot-recovery" 写入BCB
 8) erase_root() 擦除CACHE分区
 9) 清除 BCB
9. main() 调用 reboot() 重启系统

### Recovery模式流程

/init → init.rc → /sbin/recovery →
main():recovery.c

- ui_init():ui.c ［UI initialize］
- gr_init():minui/graphics.c ［set tty0 to graphic mode, open fb0]
- ev_init():minui/events.c [open /dev/input/event*]
- res_create_surface:minui/resource.c [create surfaces for all bitmaps used later, include icons, bmps]
- create 2 threads: progress/input_thread [create progress show and input event handler thread]
- get_args():recovery.c
- get_bootloader_message():bootloader.c [read mtdblock0(misc partition) 2nd page for commandline]
- check if nand misc partition has boot message. If yes, fill argc/argv.
- If no, get arguments from /cache/recovery/command, and fill argc/argv.
- set_bootloader_message():bootloader.c [set bootloader message back to mtdblock0]
- Parser argv[] filled above
- register_update_commands():commands.c [ register all commands with name and hook function ]
- registerCommand():commands.c
- Register command with name, hook, type, cookie.
- Commands, e.g: assert, delete, copy_dir, symlink, write_raw_image.
- registerFunction():commands.c
- Register function with name, hook, cookie.
- Function, e.g: get_mark, matches, getprop, file_contains
- install_package():
- translate_root_path():roots.c [ "SYSTEM:lib" and turns it into a string like "/system/lib", translate the updater.zip path ]
- mzOpenZipArchive():zip.c [ open updater.zip file (uncompass) ]
- handle_update_package():install.c
- verify_jar_signature():verifier.c [ verify signature with keys.inc key; verify manifest and zip package archive ]
- verifySignature() [ verify the signature file: CERT.sf/rsa. ]
- digestEntry():verifier.c [ get SHA-1 digest of CERT.sf file ]
- RSA_verify(public key:keys.inc, signature:CERT.rsa, CERT.sf's digest):libc/rsa.c [ Verify a 2048 bit RSA PKCS1.5 signature against an expected SHA-1 hash. Use public key to decrypt the CERT.rsa to get original SHA digest, then compare to digest of CERT.sf ]
- verifyManifest() [ Get manifest SHA1-Digest from CERT.sf. Then do digest to MANIFEST.MF. Compare them ]
- verifyArchive() [ verify all the files in update.zip with digest listed in MANIFEST.MF ]
- find_update_script():install.c [ find META-INF/com/google/android/update-script updater script ]
- handle_update_script():install.c [ read cmds from script file, and do parser, exec ]
- parseAmendScript():amend.c [ call yyparse() to parse to command ]
- exeCommandList():install.c
- exeCommand():execute.c [ call command hook function ]
- erase DATA/CACHE partition
- prompt_and_wait():recovery.c [ wait for user input: 1) reboot 2) update.zip 3) wipe data ]
- ui_key_xxx get ALT+x keys
- 1) do nothing
- 2) install_package('SDCARD:update.zip')
- 3) erase_root() → format_root_device() DATA/CACHE
- may_install_firmware_update():firmware.c [ remember_firmware_update() is called by write_hboot/radio_image command, it stores the bootloader image to CACHE partition, and write update-hboot/radio command to MISC partition for bootloader message to let bootloader update itself after reboot ]
- set_bootloader_message()
- write_update_for_bootloader():bootloader.c [ write firmware image into CACHE partition with update_header, busyimage and failimage ]
- finish_recovery():recovery.c [ clear the recovery command and prepare to boot a (hopefully working) system, copy our log file to cache as well (for the system to read), and record any intent we were asked to communicate back to the system. ]
- reboot()

### Recovery模式流程图

以下流程图绘制了系统从启动加载bootloader后的行为流程。
![](/Android-Recovery/flow1.png)
![](/Android-Recovery/flow2.png)
![](/Android-Recovery/flow3.png)

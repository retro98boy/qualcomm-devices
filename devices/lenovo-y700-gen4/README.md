# 免责声明

刷机有风险，操作需谨慎。本文提供的固件仅供交流学习使用，因刷机或使用本固件导致的任何硬件、软件或数据损坏，本人概不负责，风险由使用者自行承担

# Lenovo Y700四代的一些使用技巧

一，在平板任何状态下（操作系统运行中，操作系统卡死，EDL模式中），长按开机键加音量减键能强制重启设备。在重启过程中，合适的时机（关机后，亮屏开机前）改成按住音量加键加开机键，能让这次重启进入bootloader模式而不是boot

疑问：如果平板XBL/ABL损坏或者卡死，强制重启组合键还有用吗？

二，平板进入EDL模式的办法

- 处于安卓系统或recovery中：adb reboot edl

- 处于bootloader（fastboot）/fastbootd中：fastboot oem edl

- 接好EDL工程线再给平板上电/重启/强制重启

三，fastboot flash/boot自制boot分区镜像失败的解决措施

如果你发现fastboot flash boot_a foo.img或者fastboot boot bar.img失败（foo.img和bar.img是自制的镜像）。可以进行如下操作：

先使用fastboot flash刷回提前备份的原厂安卓分区。之前你用fastboot flash刷入了哪些自制分区就恢复哪些分区。比如你在调试系统/刷机过程中可能刷入自制的镜像到了boot_a/dtbo_a等，恢复即可。然后执行fastboot reboot recovery重启进入recovery模式（内核启动成功），最后执行adb reboot bootloader再次进入bootloader模式。这时就能成功的通过fastboot flash刷写自制的boot.img了。原理未知

PS：错误为类似下面输出
```bash
❯ fastboot boot ./Armbian-unofficial_26.11.0-trunk_Lenovo-y700-gen4_trixie_edge_7.2.11_gnome-abl_desktop.boot_sm8750-lenovo-elden.img 
< waiting for any device >
Sending 'boot.img' (87540 KB)                      OKAY [  0.203s]
Booting                                            FAILED (remote: 'Failed to load/authenticate boot image: Load Error')
fastboot: error: Command failed
```

# 系统链接

[Armbian](https://github.com/retro98boy/armbian-build)

[Batocera](https://github.com/retro98boy/batocera.linux)

[Armada](https://github.com/retro98boy/armada)

固件已知问题：

microSD卡暂时不能使用DMA，被限制成PIO，读写速度慢

屏幕暂时被固定成120Hz

不支持安卓和Linux系统共存

暂未支持的外设：

- 相机

- 麦克风

- 重力/磁力/环境光等各种传感器

# 备份全分区

强烈要求对平板进行全分区备份，因为UFS上可能存在设备独有的标定数据。而且有了全分区备份，方便在折腾的的过程中随时刷回指定的分区

首先确保平板上运行着一个完全正常的安卓系统，作者使用的是ZUXOS_1.5.10.259

备份操作使用到Windows平台上的一款名为刷机匣软件

PS：刷机匣下载链接`https://syxz.lanzoue.com/b0mbywcli`，密码bmxw。进阶Linux用户可以使用其他软件类似bkerler/edl

确保你的Windows PC安装有高通EDL驱动，将设备进入EDL模式

双击`GeekFlashTool.exe`打开刷机软件

![pictures/backup-stock-android/0.png](pictures/backup-stock-android/0.png)

点击图中红框处进入高通EDL刷机界面

![pictures/backup-stock-android/1.png](pictures/backup-stock-android/1.png)

点击图中红框处进入loader选择界面

![pictures/backup-stock-android/2.png](pictures/backup-stock-android/2.png)

选择在Windows PC上存放的loader文件，配置存储类型为UFS，然后点击确认

PS：loader文件可以在联想的EDL救砖包里面找到，名为xbl_s_devprg_ns.melf

![pictures/backup-stock-android/3.png](pictures/backup-stock-android/3.png)

点击图中红框处开始回读设备信息，包括LUN配置和分区表

![pictures/backup-stock-android/4.png](pictures/backup-stock-android/4.png)

回读分区表成功后，依次点击图中1 2 3 4处（3 4处一定要保持勾选，作者截图的时候忘了勾选）

![pictures/backup-stock-android/5.png](pictures/backup-stock-android/5.png)

经过上面1 2 3 4的勾选，你会发现所有的分区都被选中，除了metadata和userdata（如下图）。这两个分区是安卓系统用户数据，空间特别大，不推荐备份

![pictures/backup-stock-android/6.png](pictures/backup-stock-android/6.png)

点击图中红框处开始回读选中的分区

![pictures/backup-stock-android/7.png](pictures/backup-stock-android/7.png)

选择一个Windows PC的路径用于保存回读的分区镜像，不需要单独创建子目录，GeekFlashTool会自动在选中的目录下再创建子目录用于保存所有分区镜像

![pictures/backup-stock-android/8.png](pictures/backup-stock-android/8.png)

耐心等待回读完成即可

![pictures/backup-stock-android/9.png](pictures/backup-stock-android/9.png)

![pictures/backup-stock-android/10.png](pictures/backup-stock-android/10.png)

回读完成后，可使用GeekFlashTool自带的重启功能重启设备

# Linux系统刷机教程

首先需要自行搜索教程解锁BL

PS：对于该平板，理论上可以不解锁BL继续刷机。因为Y700四代早期的系统AVB使用AOSP源码中的test key签名，所以我们可以使用test key签名boot.img并搭配早期系统的ABL（早于ZUXOS_1.5.10.183版本）来引导Project Aloha。本文不讨论这部分

## EDL模式线刷（推荐，简单且快速）

这需要你按照前面的`备份全分区`章节拿到了全分区备份文件

一，获取适用于Linux系统的LUN0的GPT文件

在[Releases界面](https://github.com/retro98boy/qualcomm-devices/releases/tag/lenovo-y700-gen4)下载new-lun0-gpt.7z文件，里面是LUN0的GPT，相对于原厂安卓LUN0 GPT，减小了用于保存安卓用户数据的userdata分区大小，并增加了用于刷入Armbian/Batocera/Armada的分区

该GPT中，userdata和Linux各系统的分区大小如下：

```bash
root@lenovo-y700-gen4:~# parted /dev/sda unit GiB print
Model: MICRON MT512GBGBB4U41 (scsi)
Disk /dev/sda: 473GiB
Sector size (logical/physical): 4096B/4096B
Partition Table: gpt
Disk Flags: 

Number  Start    End      Size     File system  Name               Flags
 1      0.00GiB  0.00GiB  0.00GiB               proinfo
 2      0.00GiB  0.00GiB  0.00GiB               lenovolock
 3      0.00GiB  0.30GiB  0.29GiB  ext4         lenovocust
 4      0.30GiB  0.31GiB  0.02GiB               lenovoraw
 5      0.31GiB  0.31GiB  0.00GiB               ssd
 6      0.31GiB  0.34GiB  0.03GiB  ext4         persist
 7      0.34GiB  0.34GiB  0.00GiB               misc
 8      0.34GiB  0.34GiB  0.00GiB               keystore
 9      0.34GiB  0.35GiB  0.00GiB               frp
10      0.35GiB  22.3GiB  22.0GiB               super
11      22.3GiB  22.3GiB  0.00GiB               vbmeta_system_a
12      22.3GiB  22.3GiB  0.00GiB               vbmeta_system_b
13      22.3GiB  22.4GiB  0.06GiB               metadata
14      22.4GiB  22.7GiB  0.25GiB               rawdump
15      22.7GiB  22.8GiB  0.13GiB               pstoredump
16      22.8GiB  103GiB   80.0GiB               userdata
17      103GiB   103GiB   0.50GiB  fat32        armbian_efi        boot, esp
18      103GiB   167GiB   64.0GiB  ext4         armbian_rootfs
19      167GiB   183GiB   16.0GiB  fat32        batocera_boot      boot, esp
20      183GiB   311GiB   128GiB   ext4         batocera_userdata
21      311GiB   312GiB   1.00GiB  fat32        armada_efi         boot, esp
22      312GiB   316GiB   4.00GiB  ext4         armada_boot
23      316GiB   473GiB   156GiB   btrfs        armada_rootfs
```

其中最后一个分区armada_rootfs的大小不是固定的，对于作者512GB的Y700四代，它使用了LUN0剩下的所有空间156GiB

如果对作者提供的新LUN0 GPT不满意，觉得某个系统的分区太小了，或者不需要某个系统，也可以自己生成新LUN0 GPT

在[Releases界面](https://github.com/retro98boy/qualcomm-devices/releases/tag/lenovo-y700-gen4)下载`make-lun0-gpt.py`脚本，下载`stock-android-lun0-gpt.7z`原厂安卓LUN0 GPT并解压到某处。然后参考下面的命令重新生成LUN0 GPT

```bash
~/workspace/qualcomm/lenovo-elden
❯ ls stock-android-lun0-gpt 
gpt_backup0.bin  gpt_main0.bin  patch0.xml  rawprogram0.xml

~/workspace/qualcomm/lenovo-elden
❯ python3 make-lun0-gpt.py \
--source ./stock-android-lun0-gpt \
--output-dir ./new-lun0-gpt \
--userdata-size 80GiB \
--partition armbian_efi=512MiB:efi \
--partition armbian_rootfs=64GiB \
--partition batocera_boot=16GiB:efi \
--partition batocera_userdata=128GiB \
--partition armada_efi=1GiB:efi \
--partition armada_boot=4GiB \
--last armada_rootfs
Wrote /home/zhenlin/workspace/qualcomm/lenovo-elden/new-lun0-gpt; nominal LUN0 size: 123871232 sectors (507376566272 bytes)
  16 userdata                  5972392..26943911     (85899345920 bytes)
  17 armbian_efi              26943912..27074983     (536870912 bytes)
  18 armbian_rootfs           27074984..43852199     (68719476736 bytes)
  19 batocera_boot            43852200..48046503     (17179869184 bytes)
  20 batocera_userdata        48046504..81600935     (137438953472 bytes)
  21 armada_efi               81600936..81863079     (1073741824 bytes)
  22 armada_boot              81863080..82911655     (4294967296 bytes)
  23 armada_rootfs            82911656..123871226    (167770402816 bytes)
The final partition is resized by patch0.xml for the actual target LUN size.

~/workspace/qualcomm/lenovo-elden
❯ ls new-lun0-gpt
gpt_backup0.bin  gpt_main0.bin  patch0.xml  rawprogram0.xml
```

二，开始EDL线刷

将新LUN0 GPT文件覆盖掉全分区备份的目录里面的版本

![pictures/flash-linux/0.png](pictures/flash-linux/0.png)

确保你的Windows PC安装有高通EDL驱动，将设备进入EDL模式

双击`GeekFlashTool.exe`打开刷机软件

![pictures/flash-linux/1.png](pictures/flash-linux/1.png)

点击图中红框处进入高通EDL刷机界面

![pictures/flash-linux/2.png](pictures/flash-linux/2.png)

点击图中红框处进入loader选择界面

![pictures/flash-linux/3.png](pictures/flash-linux/3.png)

选择在Windows PC上存放的loader文件，配置存储类型为UFS，然后点击确认

PS：loader文件可以在联想的EDL救砖包里面找到，名为xbl_s_devprg_ns.melf

![pictures/flash-linux/4.png](pictures/flash-linux/4.png)

点击图中红框处来选择全分区备份目录

![pictures/flash-linux/5.png](pictures/flash-linux/5.png)

![pictures/flash-linux/6.png](pictures/flash-linux/6.png)

取消默认选择的分区，不选中任何分区

![pictures/flash-linux/7.png](pictures/flash-linux/7.png)

如下图选中LUN0的所有分区，除了metadata和userdata

![pictures/flash-linux/8.png](pictures/flash-linux/8.png)

根据需求选中Armbian/Batocera/Armada的分区，并选择对应的分区镜像。作者这里三个Linux系统都刷入，所以全选上了

![pictures/flash-linux/9.png](pictures/flash-linux/9.png)

![pictures/flash-linux/10.png](pictures/flash-linux/10.png)

选中LUN4的boot_a分区并配置镜像为Project Aloha的boot.img。Project Aloha boot.img在[Releases界面](https://github.com/retro98boy/qualcomm-devices/releases/tag/lenovo-y700-gen4)下载

![pictures/flash-linux/11.png](pictures/flash-linux/11.png)

最后点击刷入并耐心等待

![pictures/flash-linux/12.png](pictures/flash-linux/12.png)

刷入完成后，可使用GeekFlashTool自带的重启功能重启设备

## microSD卡刷机（灵活但麻烦）

一，刷入Project Aloha UEFI FD

在[Releases界面](https://github.com/retro98boy/qualcomm-devices/releases/tag/lenovo-y700-gen4)下载Project Aloha的img文件，使用fastboot flash boot_a path-to-project-aloha.img && fastboot reboot刷入即可。如果刷入失败，参照上文的“fastboot flash自制boot分区镜像失败的解决措施”

目前Project Aloha移植支持从UFS和microSD卡获取EFI App并运行，U盘暂不支持

二，从microSD卡启动系统

在提供的Armbian/Batocera/Armada的系统镜像中，.img格式的是给microSD卡使用的，直接使用写盘工具刻录到microSD卡上，插入平板再开机即可。开机后Project Aloha就会扫描microSD卡上的EFI分区，执行GRUB并引导内核。如果系统只在microSD上运行，则不用管用于刷入UFS的.xxx.img文件，也不需要看后面的教程了

三，将Linux系统刷入UFS

按照前面的教程刷入Project Aloha并从microSD卡启动Armbian，在armbian下使用parted/cfdisk等命令重新分区。该平板的整个UFS被provision成多个LUN，LUN0占用的空间最大（安卓的userdata分区也在其中）。LUN0在Armbian下被映射成/dev/sda，所以我们可以用cfdisk对/dev/sda进行分区，先将尾部的userdata删除（注意会损失安卓用户数据），再创建一个小的userdata分区，例如64G（这样安卓系统可用用户空间只有64G）。剩下的sda/LUN0空间则由我们自行分配，例如创建一个gpt part name为armbian_efi的512MiB的分区，加一个gpt part name为armbian_rootfs的64GiB的分区，加一个gpt part name为batocera_boot的16GiB的分区，加一个gpt part name为batocera_userdata的128GiB的分区。这样就能同时刷入Armbian加Batocera到UFS

PS：在EDL/fastboot模式下刷分区时指定的boot_a/userdata/abl_a等字符串是gpt part name。在Armbian下可使用类似`sudo parted /dev/sda name 18 armbian_rootfs`的命令给分区分配gpt part name。如果你重新建立了userdata分区，一定要给它分配userdata的gpt part name，安卓系统的运行依赖它

分区完毕后，直接使用dd命令将armbian-xxxx.efi.img/armbain-xxxx.rootfs.img/batocera-xxxx.boot.img/batocera-xxxx.userdata.img镜像刻录到对应的分区并重启即可

# 相关链接

[Pengyu Luo https://lore.kernel.org](https://lore.kernel.org/all/?q=f:mitltlatltl@gmail.com+AND+s:PATCH+AND+NOT+s:Re:)

[GEEKiDoS/linux](https://github.com/GEEKiDoS/linux)

[h0cheung/tb322fc-linux-kernel](https://github.com/h0cheung/tb322fc-linux-kernel)

[xiaquwan1001/lenovo-legion-y700-gen4-tb322fc-elden-linux](https://github.com/xiaquwan1001/lenovo-legion-y700-gen4-tb322fc-elden-linux)

---
title: 调整VirtualBox CentOS虚拟机磁盘大小
date: 2017-05-10T12:54:24+08:00
tags:
- VirtualBox
- CentOS
image: vbox.png
---

# 背景
昨天发现本地HDP集群的HBase连不上了，解决了这个问题后，又发现HBase同步脚本一直执行失败，进YARN看了下，几天前开始堆积了很多ACCEPT了的任务，但没一个在执行，先用```hadoop job kill ```干掉了旧的任务，重新执行脚本，还是不行，日志上面却没报错。  
于是进Ambari一看，HDFS的DataNode全都满了……看来要扩容了。  
集群是部署在几台VirtualBox的CentOS虚拟机上的（起始我一直想转移到Docker上，但一直没时间+懒）。网上关于VirtualBox虚拟机磁盘扩容的文章很多，试了下，很多是行不通的，而且不太完整，于是把今天的经验记下来吧。

# 具体步骤
## 磁盘映像扩容
下面的命令以名为CentOS05的虚拟机为例进行。  
关闭虚拟机。
```bash
vboxmanage controlvm CentOS05 poweroff
```
因为原来的磁盘是vmdk的不能直接扩展容量，需要先转换成vdi(这个步骤相当缓慢，视乎电脑配置和原磁盘映像大小，请耐心等待)：  
```bash
cd /path/to/虚拟机vmdk磁盘映像存储位置
vboxmanage clonehd CentOS05-disk1.vmdk CentOS05-disk1.vdi --format VDI
```
再扩容，注意默认单位是MB：
```bash
vboxmanage modifyhd CentOS05-disk1.vdi --resize 122880
```

## 修改虚拟机挂载磁盘
查看虚拟机原来配置
```bash
VBoxManage showvminfo CentOS05
```
注意类似以下的信息，包括名称（IDE），端口号，设备ID（如IDE(1,0)的端口号是1，设备号0），下面要用到：
> Storage Controller Name (0):            IDE
Storage Controller Type (0):            PIIX4
Storage Controller Instance Number (0): 0
Storage Controller Max Port Count (0):  2
Storage Controller Port Count (0):      2
Storage Controller Bootable (0):        on
IDE (0, 0): /home/turing/VirtualBox VMs/CentOS03/CentOS03-disk1.vdmk (UUID: 2e2d32dc-e66b-42c2-8c9c-4cbb7abb6182)
IDE (1, 0): Empty

修改虚拟机挂载的磁盘：
```bash
VBoxManage storageattach CentOS05 --storagectl 'IDE' --port 0 --device 0 --type hdd --medium CentOS05-disk1.vdi 
```
参数说明：
> storagectl：存储控制器的名称。必须。
port：介质将被连接／断开／修改的端口号。必须。
device：介质将被连接／断开／修改的设备号。必须。
type：定义 介质将被连接／断开／修改的驱动器类型。
medium：允许指定DVD／软盘驱动器是完全断开的（none）或仅是需要被连接的空的DVD／软盘驱动器（emptydrive）。如果指定了uuid，filename或host:<drive>，将连接到存储控制器的指定端口和设备号。

再次查看配置确认修改成功：
```bash
VBoxManage showvminfo CentOS05
```

## 修改CentOS分区配置
因为在系统启动后根目录的卸载之类的动作旧就不能执行了，所以只能在LiveCD里面改，建议使用gparted图形化工具修改分区大小，比较简单，此处不表。  
修改好启动虚拟机：
```bash
vboxmanage startvm CentOS05 --type headless   
ssh CentOS05
```
查看分区情况：
```bash
fdisk -l /dev/sda
```
>Disk /dev/sda: 128.8 GB, 128849018880 bytes
255 heads, 63 sectors/track, 15665 cylinders
Units = cylinders of 16065 * 512 = 8225280 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disk identifier: 0x000636d5
   Device Boot      Start         End      Blocks   Id  System
/dev/sda1   *           1          64      512000   83  Linux
Partition 1 does not end on cylinder boundary.
/dev/sda2              64       15666   125316096   8e  Linux LVM

```bash
df -h
```
>Filesystem            Size  Used Avail Use% Mounted on
/dev/mapper/vg_centos03-lv_root
                       50G   44G  3.1G  94% /
tmpfs                 1.9G     0  1.9G   0% /dev/shm
/dev/sda1             477M   30M  422M   7% /boot
/dev/mapper/vg_centos03-lv_home
                      6.4G   82M  6.0G   2% /home

发现fdisk正确识别分区容量，但df还是旧容量。仔细观察df的输出，原来LVM管理的，需要执行修改逻辑卷的命令。先看情况：
```bash
pvdisplay -v -m
```
>Using physical volume(s) on command line.
  Wiping cache of LVM-capable devices
  Finding all volume groups.
--- Physical volume ---
PV Name               /dev/sda2
VG Name               vg_centos03
PV Size               119.51 GiB / not usable 2.00 MiB
Allocatable           yes 
PE Size               4.00 MiB
Total PE              30594
Free PE               15360
Allocated PE          15234
PV UUID               Eq5aod-THPs-vdcM-T40A-fCQW-dqQ1-NERrdH
--- Physical Segments ---
Physical extent 0 to 12799:
  Logical volume      /dev/vg_centos03/lv_root
  Logical extents     0 to 12799
Physical extent 12800 to 14477:
  Logical volume      /dev/vg_centos03/lv_home
  Logical extents     0 to 1677
Physical extent 14478 to 15233:
  Logical volume      /dev/vg_centos03/lv_swap
  Logical extents     0 to 755
Physical extent 15234 to 30593:
  FREE

结合```df -h```的输出，可知我们要改的是就是vg_centos03/lv_root逻辑卷，修改容量为110G：
```bash
lvresize -L 110G -r vg_centos05/lv_root
```
再次查看容量发现OK。  
进入Ambari，重启服务，也看到空间警告消除了。
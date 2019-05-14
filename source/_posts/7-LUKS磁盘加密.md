---
title: LUKS磁盘加密
date: 2018-08-13 23:52:13
tags:
 - storage
---
### 前言
LUKS （Linux Unified Key Setup）是 Linux 硬盘加密的标准。通过提供标准的磁盘格式，它不仅可以促进发行版之间的兼容性，还可以提供对多个用户密码的安全管理。 与现有解决方案相比，LUKS 将所有必要的设置信息**存储在分区信息**首部中，使用户能够无缝传输或迁移其数据。

- 只有在开启映射的时候需要密码，在映射分区的挂载和使用是不需要密码的。
- 加密后分区不能直接挂载，只能挂载映射分区，一般用来防止磁盘在其他机器上访问。
- 支持 LVM，LVM 和 dm-crypt 都是基于 Linux 内核的 device mapper 机制。



### LUKS 的使用步骤

```c
// 1.正常分区
$ parted / fstab      

// 2.对为初始化的分区进行加密，输入密码或者使用文件内容加密,可选加密方式
$ cryptsetup luksFormat /dev/sdc1（/dev/VG/LV）  
$ cryptsetup --cipher aes-xts-plain64 --key-size 512 --hash sha512 --iter-time 10000 luksFormat /dev/sda2

    //密码放在文件内
    $ cryptsetup luksFormat DEV /root/random_data_keyfile1
    //增加密码
    $ cryptsetup luksAddKey /dev/sdc1 /passwd_txt

// 3.打开加密的分区，将其映射到一个不加密的分区上   /dev/mapper/luks-uuid分区
$ cryptsetup luksOpen /dev/sdc1 luks--$(cryptsetup luksUUID /dev/sda3)

// 4.格式化分区并挂载
$ mkfs.ext4 /dev/mapper/test


// 5.umount 后 可以进行 取消映射
$ cryptsetup luksClose test

```

### 开机自动识别加密磁盘

/etc/crypttab  文件记录映射内容以及解密密码所在文件。

```c
$ cat /etc/crypttab 
test /dev/sdc1  /passwd_txt
xxx   /dev/sda15   /root/keyfile   luks

// (映射名，分区，key文件位置，最后一列就是'luks')
```

#### 根分区自动识别
需要在 grub 中声明 LUKS 加密的磁盘以及解密文件

```c
//rhel6
$ sed -i "/^\s*kernel/s,$, rd_LUKS_UUID=$(cryptsetup luksUUID /dev/sda3)," /boot/grub/grub.conf   

// rhel7
$ sed -i "/^GRUB_CMDLINE_LINUX/s,\"$, rd.luks.uuid=$(cryptsetup luksUUID /dev/sda3)\"," /etc/default/grub  
$ grub2-mkconfig >/etc/grub2.cfg 

$ dracut -f

```

### 忘记密码怎么办？

 1. 设置多个密码的情况，可以尝试找到其他密码
 ```
 $ blkid -t TYPE=crypto_LUKS -o device
 /dev/vdb1
 
 $ cryptsetup luksDump /dev/vdb1 | grep Key.Slot
 Key Slot 0: ENABLED
 Key Slot 1: DISABLED
 Key Slot 2: DISABLED
 Key Slot 3: DISABLED
 Key Slot 4: DISABLED
 Key Slot 5: DISABLED
 Key Slot 6: DISABLED
 Key Slot 7: DISABLED
 
 ```
 2. 如果磁盘当前可用，可以使用 root 用户通过master key增加新的密码
 ```
 //首先找到设备，使用lsblk, findmnt, df, mount, or /etc/fstab确认
 $ dmsetup ls --target crypt
 vdc-decrypted    (253, 2)
 luks-ec013cf7-ad72-4dcf-8a1e-0548016a3e2c    (253, 1)
 
 //找到 master key ,在第五列
 $ dmsetup table <MAP> --showkeys
 
 //通过master key增加新的密码,需要先转换成binary
 $ lsblk | grep -B1 luks-ec013cf7-ad72-4dcf-8a1e-0548016a3e2c
   └─vdb1                                        252:17   0 1023M  0 part
     └─luks-ec013cf7-ad72-4dcf-8a1e-0548016a3e2c 253:1    0 1021M  0 crypt /cryptstor
 $ cryptsetup luksAddKey /dev/vdb1 --master-key-file <(dmsetup table --showkey  /dev/mapper/luks-ec013cf7-ad72-4dcf-8a1e-0548016a3e2c | awk '{print$5}' | xxd -r -p)
 Enter new passphrase for key slot:
 Verify passphrase:
 $ cryptsetup luksDump /dev/vdb1 | grep ENABLED
 Key Slot 0: ENABLED
 Key Slot 1: ENABLED
 ```

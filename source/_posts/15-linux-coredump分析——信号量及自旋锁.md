---
title: linux coredump 分析——信号量及自旋锁
date: 2019-05-31 19:54:11
tags:
 - kernel
 - core dump
---

## 一、前言
当 linux 系统发生 hung 死，并在日志中能看到 `softlockup` 、`hung_task` 等信息时，或者机器自己发生 panic 死机时，我们可以通过配置 kdump 捕获内核转储，然后使用 crash 工具分析内核因为什么 hung 死，并死在哪一个步骤。


> kdump 是一种先进的基于 kexec 的内核崩溃转储机制。当系统崩溃时，kdump 使用 kexec 启动到第二个内核。第二个内核通常叫做捕获内核，以很小内存启动以捕获转储镜像。第一个内核保留了内存的一部分给第二内核启动用。由于 kdump 利用 kexec 启动捕获内核，绕过了 BIOS，所以第一个内核的内存得以保留。这是内核崩溃转储的本质。

kdump 工作步骤如下：

![kdump](https://raw.githubusercontent.com/Tomoku-dm/blog-images/master/15-kdump.jpg)
## 二、Core dump 收集

1. 修改内核参数使其 panic
    ```shell
    $ cat /etc/sysctl.conf
    kernel.hung_task_panic = 1
    kernel.softlockup_panic = 1
    kernel.sysrq = 0
    
    $ sysctl -p
    ```

2. 安装 kdump ，并设置为第二内核预留的内存

- **Centos6**
    ```shell
    $ yum install kexec-tools
    
    $ cat /boot/grub/grub.conf               ## 2G 以上内存设置 auto，2G 以下设置 128M 。配置完需要重启生效
    kernel /vmlinuz-2.6.32-431.el6.x86_64 ro root=UUID=638db084-d800-4875-bde8-d065e97799be rd_NO_LUKS rd_NO_LVM LANG=en_US.UTF-8 rd_NO_MD SYSFONT=latarcyrheb-sun16 crashkernel=auto  KEYBOARDTYPE=pc KEYTABLE=us rd_NO_DM rhgb quiet crashkernel=auto
    
    # cat /sys/kernel/kexec_crash_size            ## 查看 kdump 预留的内存
    143654912
    
    $ chkconfig kdump on && service kdump start
    $ echo c > /proc/sysrq-trigger                ## 模拟panic ，测试是否生成vmcore,默认生成在/var/crash下,希望存到其他存储看参考链接
    ```
- **Centos7**
    ```shell
    $ yum install kexec-tools
    
    # cat /etc/default/grub           ## 2G 以上内存设置 auto，2G 以下设置 128M 。配置完需要重启生效
    GRUB_TIMEOUT=5
    GRUB_DISTRIBUTOR="$(sed 's, release .*$,,g' /etc/system-release)"
    GRUB_DEFAULT=saved
    GRUB_DISABLE_SUBMENU=true
    GRUB_TERMINAL_OUTPUT="console"
    GRUB_CMDLINE_LINUX="crashkernel=auto rd.lvm.lv=centos/root rd.lvm.lv=centos/swap rhgb quiet crashkernel=auto"
    GRUB_DISABLE_RECOVERY="true"
    
    # cat /sys/kernel/kexec_crash_size            ## 查看 kdump 预留的内存
    143654912
    
    $ systemctl start kdump && $ systemctl enable kdump
    $ echo c > /proc/sysrq-trigger                ## 模拟panic ，测试是否生成vmcore,默认生成在/var/crash下,希望存到其他存储看参考链接
    ```



## 三、vmcore 分析环境部署

```shell
$ yum -y install crash
$ yum -y install kernel-debuginfo*

## 没有debug 的 yum 源可以去下面的链接下载，找当前使用内核对应的版本包
## http://debuginfo.centos.org/7/x86_64/

# uname -rsp                          ## 下载debuginfo 包要和vmcore 抓的内核版本统一
Linux 3.10.0-693.el7.x86_64 x86_64
$ wget http://debuginfo.centos.org/7/x86_64/kernel-debuginfo-3.10.0-693.el7.x86_64.rpm
$ wget http://debuginfo.centos.org/7/x86_64/kernel-debuginfo-common-x86_64-3.10.0-693.el7.x86_64.rpm

$ rpm -ivh kernel-debuginfo-3.10.0-693.el7.x86_64.rpm kernel-debuginfo-common-x86_64-3.10.0-693.el7.x86_64.rpm

## 如果需要使用dubuginfo-install 安装其他包需提前安装
$ yum install nss-softokn-debuginfo --nogpgcheck
$ yum install yum-utils

```
在分析过程中可能需要查看内核源码，在[这里](http://vault.centos.org/6.5/os/Source/SPackages/)可以下载到 Centos 各种包的源码
```shell
## 把packName.src.rpm解包，会生成一个.tar.gz或者.tar.bz2的压缩包，那个就是源码
$ rpm2cpio packName.src.rpm | cpio -id
##  解压缩源码包
$ tar -jxvf packName.tar.bz(2)
```

## vmcore 分析案例
下面是一个关于信号量读写锁的 vmcore 分析案例, 本案例相对比较复杂，设计 `信号量锁` 以及 `spinlock 自旋锁`，下面我们分两个阶段来进行排查

### `信号量锁` 阶段
1. 首先查看处于 UN 状态的进程数量

   ```shell
   crash> ps | grep UN | wc -l
   94
   crash> ps | grep RU | wc -l
   43
   crash> ps | grep IN | wc -l
   2468
   crash> ps -u |grep UN
     30413  30364  24  ffff88022e66cae0  UN   0.8 9333496 1128356  java
     30422  30364  23  ffff88022e7b6aa0  UN   0.8 9333496 1128356  java
     30608  30553  28  ffff88039e084040  UN   0.8 9334548 1036756  java
     30681  30553  27  ffff881c3dc8c040  UN   0.8 9334548 1036756  java
     30725  30553   4  ffff8801dc940ae0  UN   0.8 9334548 1036756  java
     30726  30553   9  ffff88039c94a040  UN   0.8 9334548 1036756  java
     30767  30553  28  ffff88022e444aa0  UN   0.8 9334548 1036756  java
     31734      1  27  ffff88039e33e040  UN   0.0       0      0  java
     31735      1  26  ffff881b2ef2c080  UN   0.0       0      0  java
     31736      1  16  ffff881f8e84cae0  UN   0.0       0      0  java
     31738      1   8  ffff881c8c812040  UN   0.0       0      0  java
     ...
     31739      1  13  ffff881d14f10aa0  UN   0.0       0      0  java
     31740      1  24  ffff881c8c812aa0  UN   0.0       0      0  java
     31855      1   5  ffff88066c18e040  UN   0.0       0      0  java
     31856   5427  19  ffff881b18d40040  UN   0.0    6604   3200  pidof
     31858  31857  26  ffff881c87c4e040  UN   0.0  108144    988  ps
     31908  31907   8  ffff882050d65540  UN   0.0  108152   1020  ps
   ```

2. 选择一个进程 ，可以看到这个进程在等待信号量锁，然后我们找到信号量锁的地址

   ```shell
   ## 选择一个等待信号量锁的进程
   crash> bt 30413
   PID: 30413  TASK: ffff88022e66cae0  CPU: 24  COMMAND: "java"
    #0 [ffff88031bdebd58] schedule at ffffffff815278c2
    #1 [ffff88031bdebe20] rwsem_down_failed_common at ffffffff81529f85
    #2 [ffff88031bdebe80] rwsem_down_write_failed at ffffffff8152a0e3
    #3 [ffff88031bdebec0] call_rwsem_down_write_failed at ffffffff8128e883
    #4 [ffff88031bdebf20] sys_mprotect at ffffffff81152386
    #5 [ffff88031bdebf80] system_call_fastpath at ffffffff8100b072
       RIP: 000000303a8e5677  RSP: 00007fb3cd03d3d0  RFLAGS: 00010202
       RAX: 000000000000000a  RBX: ffffffff8100b072  RCX: 0000000000000000
       RDX: 0000000000000001  RSI: 0000000000001000  RDI: 00007fb3e2ce9000
       RBP: 00007fb3cd03d990   R8: 00007fb3e2a99d80   R9: 0000000000000001
       R10: 0000000000000000  R11: 0000000000000206  R12: 00007fb3e2a995e8
       R13: 00007fb3dc0a4000  R14: 00007fb3e2a903b0  R15: 0000000000001000
       ORIG_RAX: 000000000000000a  CS: 0033  SS: 002b
   
   
   ## 拿到这个进程 mm.struct 中 信号量锁的地址
   crash> p &((struct task_struct *)0xffff88022e66cae0)->mm->mmap_sem
   $5 = (struct rw_semaphore *) 0xffff8805bdec27e8
   ```

3. 信号量锁的地址找到了，我们先研究一下信号量锁相关的内容

   ```shell
   ## 拿到进程 mm_struct 的地址
   crash> task_struct.mm ffff88022e66cae0
     mm = 0xffff8805bdec2780
     
   ## 看看 mm_struct 的 mmap_sem 存放的都是什么
   crash> mm_struct.mmap_sem 0xffff88022e66cae0
     mmap_sem = {
       count = -131932026909888,             ## 这个count 是两个16位计数器 需要计算得知持有锁进程的数量
       wait_lock = {
         raw_lock = {
           slock = 0
         }
       }, 
       wait_list = {
         next = 0x81f905286d6, 
         prev = 0x57d6bdb
       }
     }
     
   ## 我们也可以直接拿到 count
   crash>  p  (((struct mm_struct *) 0xffff88022e66cae0)->mmap_sem).count 
   $6 = -131932026909888
   
   
   ## 我们在mmap_sem 中可以看到有wait_list 队列，那我们看看队列里都有哪些进程在等这个信号量锁
   
   ## 首先要拿到这些进程 的 父进程
   crash> px ((struct task_struct *)0xffff88022e66cae0)->mm.owner
   $7 = (struct task_struct *) 0xffff88063f270080
   
   crash> px ((struct task_struct *)0xffff88063f270080)->mm->mmap_sem
   $8 = {
     count = 0xfffffffd00000001, 
     wait_lock = {
       raw_lock = {
         slock = 0x16b016b
       }
     }, 
     wait_list = {
       next = 0xffff88031bdebe88, 
       prev = 0xffff88039aac7d60
     }
   }
   
   ## 从下面结果可以看到 一共有三个进程在等
   crash> list -H 0xffff88031bdebe88 -o rwsem_waiter.list -s rwsem_waiter.task 
   ffff881c8cb0dd80
     task = 0xffff882050d65540
   ffff88039aac7d60
     task = 0xffff88022e7b6aa0
   ffff8805bdec27f8
     task = 0xcce0cce
   
   
   ```

4. 接下来我们继续，拿到信号量锁结构体的地址，我们就可以查哪个进程拿着这个说没有释放

   ```shell
   ## 通过foreach 遍历所有带这个地址的进程，看哪个进程像
   crash> foreach bt -f  | grep -B60  ffff8805bdec27e8    ## 60 行不够展示的话  适当增加
   
   ## 从结果中大概有三类进程
   PID: 31908  TASK: ffff882050d65540  CPU: 8   COMMAND: "ps"
    #0 [ffff881c8cb0dc50] schedule at ffffffff815278c2
    #1 [ffff881c8cb0dd18] rwsem_down_failed_common at ffffffff81529f85
    #2 [ffff881c8cb0dd78] rwsem_down_read_failed at ffffffff8152a116
    #3 [ffff881c8cb0ddb8] call_rwsem_down_read_failed at ffffffff8128e854
    #4 [ffff881c8cb0de20] m_start at ffffffff811f1934
    #5 [ffff881c8cb0de70] seq_read at ffffffff811add56
    #6 [ffff881c8cb0def0] vfs_read at ffffffff811896a5
    #7 [ffff881c8cb0df30] sys_read at ffffffff811897e1
    #8 [ffff881c8cb0df80] system_call_fastpath at ffffffff8100b072
   PID: 30902  TASK: ffff881b693cd540  CPU: 24  COMMAND: "java"
    #0 [ffff881b18ea5b38] schedule at ffffffff815278c2
    #1 [ffff881b18ea5c00] futex_wait_queue_me at ffffffff810ae8d9
    #2 [ffff881b18ea5c40] futex_wait at ffffffff810af9e8
    #3 [ffff881b18ea5dc0] do_futex at ffffffff810b1151
    #4 [ffff881b18ea5ef0] sys_futex at ffffffff810b1c0b
    #5 [ffff881b18ea5f80] system_call_fastpath at ffffffff8100b072
   PID: 30388  TASK: ffff88039e30b500  CPU: 20  COMMAND: "java"
    #0 [ffff880028387e90] crash_nmi_callback at ffffffff8102fee6
    #1 [ffff880028387ea0] notifier_call_chain at ffffffff8152d515
    #2 [ffff880028387ee0] atomic_notifier_call_chain at ffffffff8152d57a
    #3 [ffff880028387ef0] notify_die at ffffffff810a154e
    #4 [ffff880028387f20] do_nmi at ffffffff8152b1db
    #5 [ffff880028387f50] nmi at ffffffff8152aaa0
       [exception RIP: _spin_lock_irq+40]
       RIP: ffffffff8152a238  RSP: ffff88049d8b5638  RFLAGS: 00000093
       RAX: 00000000000066ce  RBX: ffff880000022e80  RCX: 0000000000000004
       RDX: 00000000000066cc  RSI: 000000000000000a  RDI: ffff88000002b2c0
       RBP: ffff88049d8b5638   R8: 000000000000000a   R9: 0000000000000000
       R10: 0000000000000000  R11: 0000000000000004  R12: ffffea001a8688b8
       R13: 0000000000000000  R14: ffff8800283931c0  R15: 0040000000020028
       ORIG_RAX: ffffffffffffffff  CS: 0010  SS: 0018
   --- <NMI exception stack> ---
    #6 [ffff88049d8b5638] _spin_lock_irq at ffffffff8152a238
    #7 [ffff88049d8b5640] ____pagevec_lru_deactivate at ffffffff81136cc8
    #8 [ffff88049d8b56d0] lru_add_drain at ffffffff81136f68
    #9 [ffff88049d8b5700] __pagevec_release at ffffffff81136fd6
   #10 [ffff88049d8b5720] invalidate_mapping_pages at ffffffff811375b8
   #11 [ffff88049d8b5810] shrink_icache_memory at ffffffff811a602b
   #12 [ffff88049d8b5870] shrink_slab at ffffffff81138ada
   #13 [ffff88049d8b58d0] zone_reclaim at ffffffff8113b6de
   #14 [ffff88049d8b59f0] get_page_from_freelist at ffffffff8112d83c
   #15 [ffff88049d8b5b20] __alloc_pages_nodemask at ffffffff8112f3a3
   #16 [ffff88049d8b5c60] alloc_pages_vma at ffffffff81167b9a
   #17 [ffff88049d8b5cb0] handle_pte_fault at ffffffff8114ac9d
   #18 [ffff88049d8b5d90] handle_mm_fault at ffffffff8114b28a
   #19 [ffff88049d8b5e00] __do_page_fault at ffffffff8104a8d8
   #20 [ffff88049d8b5f20] do_page_fault at ffffffff8152d45e
   #21 [ffff88049d8b5f50] page_fault at ffffffff8152a815
   
   ## 前两种是没有拿到锁的，do_futex 后面 调用了 futex_wait 可知发生了信号量锁互斥。最后一个在 __do_page_fault 中拿锁，然后卡在 _spin_lock_irq 上了。
   ```



### `自旋锁`阶段

接下来我们就分析 PID: 30388 进程 为什么没有拿到 spinlock 。

通常来说，对于类似的死锁问题分析，都会有这种思路：死锁，那就肯定有进程持有相应的锁而一直不释放，导致本进程一直获取不到锁。那就需要寻找持有锁的进程了.

持有该锁的进程可能处于如下几种状态(按可能性大小排列)：

1. 处于RUNNING状态，且正在其它某CPU上运行。
2. 处于RUNNING状态，但暂时没有得到调度运行。
3. 处于D状态，等待某任务完成。
4. 处于S状态
5. 已经运行结束，进程已经不存在。

最可能的肯定是1、2、3。 4 和 5  属于明显的内核bug，对于 3 和 4来说，因为`spinlock`持有者是不能sleep的，内核中对于持有`spinlock`再进行 sleep 的情况应该有判断和告警；对于5来说，那就是有进程持有`spinlock`锁，但没有释放就退出了(比如某些异常分支)，相当于泄露，这种情况内核中有静态代码检查工具，对于一般的用户态程序这种错误可能容易出现，但内核中这种可能性也极小。



照这种思路，分析过程大致为：



```shell
bt 30388
 #6 [ffff88049d8b5638] _spin_lock_irq at ffffffff8152a238
 #7 [ffff88049d8b5640] ____pagevec_lru_deactivate at ffffffff81136cc8
 #8 [ffff88049d8b56d0] lru_add_drain at ffffffff81136f68
 
crash> dis -l  ____pagevec_lru_deactivate
## rdi 是 传入的参数  spinlock

crash> spinlock_t ffff88000002b2c0
struct spinlock_t {
  raw_lock = {
    slock = 1725195980
  }
}
crash> eval 1725195980
hexadecimal: 66d466cc  
    decimal: 1725195980  
      octal: 14665063314
     binary: 0000000000000000000000000000000001100110110101000110011011001100
## 4 字节的lock分成两部分：
## Next(2字节)|Owner(2字节)
## 0110011011010100    20
## 0110011011001100    12     说明有8 个在等锁

bt -a   看到 cpu 上除了swapper 就是 等自旋锁
PID: 1514   TASK: ffff88066065a040  CPU: 15  COMMAND: "java"
PID: 1468   TASK: ffff8805d96c6aa0  CPU: 19  COMMAND: "java"


## 看 spinlock 到哪一步了
dis -l ffffffff8152a238
crash> dis -l  ffffffff8152a238
/usr/src/debug/kernel-2.6.32-431.el6/linux-2.6.32-431.el6.x86_64/arch/x86/include/asm/spinlock.h: 127
0xffffffff8152a238 <_spin_lock_irq+40>: jmp    0xffffffff8152a22f <_spin_lock_irq+31>
crash> l /usr/src/debug/kernel-2.6.32-431.el6/linux-2.6.32-431.el6.x86_64/arch/x86/include/asm/spinlock.h: 127
122     static __always_inline void __ticket_spin_lock(raw_spinlock_t *lock)
123     {
124             int inc;
125             int tmp;
126     
127             asm volatile("1:\t\n"
128                          "mov $0x10000, %0\n\t"
129                          LOCK_PREFIX "xaddl %0, %1\n"
130                          "movzwl %w0, %2\n\t"
131                          "shrl $16, %0\n\t"


## 看哪里调用的 这个spinlock  会不有又被人同时加锁
dis -l ffffffff81136cc8
```



下面为什么没有了？？？   因为看不懂了。。。





## crash 常用命令
- crash /
- bt TAKSID
- dis -rl  taskID
- ps 
- set
- foreach bt -f 
- sys
- kmem -i


## 参考链接
 - [RedHat: 如何使用 kdump 服务来分析内核崩溃，系统挂起，系统重启？](https://access.redhat.com/zh_CN/solutions/667083)
 - [RedHat: How should the crashkernel parameter be configured for using kdump on RHEL7?](https://access.redhat.com/solutions/916043)
 - [IBM: 深入探索 Kdump](https://www.ibm.com/developerworks/cn/linux/l-cn-kdump1/index.html)
 - [用crash工具分析Linux内核死锁的一次实战](https://blog.csdn.net/juS3Ve/article/details/79428049)
 - [读写信号量与实时进程阻塞挂死问题](http://oenhan.com/rwsem-realtime-task-hung)
 - [futex 相关](https://blog.csdn.net/csq_year/article/details/49304477)
 - https://bugzilla.redhat.com/show_bug.cgi?id=669418
 - https://blog.csdn.net/u010720841/article/details/39579515
 - https://access.redhat.com/solutions/1593753
 - https://access.redhat.com/support/cases/#/case/02133153
 - https://access.redhat.com/support/cases/#/case/01883655
 

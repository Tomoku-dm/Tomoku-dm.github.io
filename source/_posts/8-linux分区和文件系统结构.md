---
title: linux 分区和文件系统结构
date: 2018-08-19 00:17:31
tags:
- storage
- filesystem
---
## 前言

本篇文章总结一下磁盘分区以及文件系统的结构，以及文件的 inode 、文件描述符(file descriptor)的用法和概念。

## 磁盘分区  
关于磁盘的物理结构不做太多描述，主要讲述分区的细节，以及第一个扇区上的引导。

**磁盘物理结构**：
- ![8-disk](https://raw.githubusercontent.com/Tomoku-dm/blog-images/master/8-disk.jpeg)
- 磁头（Head）：代表有多少层盘面
- 磁道（Track）
- 柱面（Cylinder）：每一个盘面上有多少圈磁道
- 扇区（Sector）：一圈磁道上有很多扇区，每个扇区有512字节，第一个扇区称为引导扇区。
- fdisk 可以看到每一块盘上述结构的数量。fdisk 看到磁盘的大小 = （盘面 * 柱面 * 扇区 * 512bytes）

### 1.分区结构
磁盘通过分区，可以实现多个文件系统，多个操作系统，快速的IO。物理上分区是通过柱面来进行分割的。目前最为广泛的还是MBR分区，但是 MBR 最大只支持2T 磁盘，且只支持4个主分区，所以未来将逐渐使用 GPT 分区。

#### MBR 分区
MBR (Master Boot Record)  是指磁盘第一个扇区，包括引导程序、MBR分区表、MBR结束标志3部分构成，一共占用512个字节。

![8-mbr](https://raw.githubusercontent.com/Tomoku-dm/blog-images/master/8-mbr.jpg)


- 主引导程序(boot loader)，446字节，有grub(linux),LILO,BOOTMGR等，提供给用户可以启动的操作系统，**每一个分区开头也有一个引导程序部分**，通过主引导找到分区上的引导程序可以实现多系统。
- 分区表，64字节，每一个分区需要16个字节。MBR 只能有4个主分区，或者3+1扩展分区。
    - 每一个分区表16字节中有4个字节来记录扇区号，每个分区最大支持2T，2^(4*8)*512byte=2T，通过4K大小的扇区可以解决。
- 结束标志2字节，55aa = (0101 0101 1010 1010 )


#### GPT 分区
GPT 分区（GUID Partition Table）全称全局唯一标识分区表，是EFI标准一部分，用来代替bios以及MBR。GPT分区使用64bit记录逻辑地址，8 ZiB个扇区大小的磁盘。

![8-gpt](https://raw.githubusercontent.com/Tomoku-dm/blog-images/master/8-gpt.jpeg)
 
- GPT 使用逻辑区块地址（LBA  512字节一个，一个扇区大小）取代了早期的CHS寻址方式。在非传统硬盘上，例如SSD 有可能LBA 是2K/4K。
- 传统 MBR 信息存储于LBA 0，引导启动程序也在这里，GPT 头存储于LBA 1，接下来才是分区表本身。
- GPT 分区表大小不是固定的，64位Windows操作系统使用16,384字节（或32扇区）作为GPT分区表(**最多128个分区**=16,384/128)，
- 接下来的LBA 34是硬盘上第一个分区的开始。


## 文件系统

### 1.文件系统结构
![8-partition](https://raw.githubusercontent.com/Tomoku-dm/blog-images/master/8-partition.png)


- Boot Sector ：每个分区最前面都有一个引导区，被主引导找到。
- Super Block(超级块): 存放文件系统的结构信息, 说明各部分的大小. `super_block[8]`, 可加载8个文件系统
- Inode Bitmap(i节点位图): 记录i节点的使用情况, 1bit代表一个i节点. `s_imap[8]`, 占用8个块, 可表示8191个i节点情况
- Block Bitmap(逻辑块位图): 记录数据区的使用情况, 1bit代表一个盘块(block). `s_zmap[8]`, 占用8个块, 最大支持64M的硬盘
- Inode(i节点) Table: 存放inode 号和数据块的对应关系。每个文件或目录名唯一对应一个i节点, 在i节点中, 储存 id信息, 文件长度, 时间信息, 实际数据所在位置等等。**目录文件的data block 中存放的是目录下文件名和对应的 inode 号。**


### 2.inode 如何找到文件。
在Linux中，我们通过解析路径，根据沿途的目录文件来找到某个文件。目录中的条目除了所包含的文件名，还有对应的inode编号。

当我们输入$cat /var/test.txt时，Linux 将在根目录文件中找到 var 这个目录文件的inode编号，然后根据 inode 合成 var 的数据。随后，根据 var 中的记录，找到 text.txt 的 inode 编号，沿着 inode 中的指针，收集数据块，合成 text.txt 的数据。

整个过程中，会参考三个inode：  
1. 根目录文件的 inode：2，用于找到 var 的 inode id  
2. var 目录文件的 inode：10747905，用于找到 test.txt 的 inode id  
3. text.txt 文件的 inode： 10749034，用于找到 data blocks

![8-inode-select](https://raw.githubusercontent.com/Tomoku-dm/blog-images/master/8-inode-select.png)


### 3.File descriptor
对于linux而言，所有对设备和文件的操作都使用文件描述符来进行的。文件描述符是一个非负的整数，它是一个索引值，指向内核中每个进程打开文件的记录表。当打开一个现存文件或创建一个新文件时，内核就向进程返回一个文件描述符；当需要读写文件时，也需要把文件描述符作为参数传递给相应的函数。 
通常，一个进程启动时，都会打开3个文件：标准输入、标准输出和标准出错处理。这3个文件分别对应文件描述符为0、1和2（宏STD_FILENO、STDOUT_FILENO和STDERR_FILENO）。

**每一个文件描述符会与一个打开文件相对应**，同时，不同的文件描述符也会指向同一个文件。相同的文件可以被不同的进程打开也可以在同一个进程中被多次打开。系统为每一个进程维护了一个文件描述符表，该表的值都是从0开始的，所以在不同的进程中你会看到相同的文件描述符，这种情况下相同文件描述符有可能指向同一个文件，也有可能指向不同的文件。具体情况要具体分析，要理解具体其概况如何，需要查看由内核维护的3个数据结构。 
1. 进程级的文件描述符表 
2. 系统级的打开文件描述符表 
3. 文件系统的i-node表

由于进程级文件描述符表的存在，不同的进程中会出现相同的文件描述符，它们可能指向同一个文件，也可能指向不同的文件。两个不同的文件描述符，若指向同一个打开文件句柄，将共享同一文件偏移量。因此，如果通过其中一个文件描述符来修改文件偏移量，那么从另一个文件描述符中也会观察到变化，无论这两个文件描述符是否属于不同进程，还是同一个进程，情况都是如此。

进程通过文件描述符、文件系统表（打开的文件句柄 ） 以 及  i-node之间的关系找到对应文件。  
![8-fd](https://raw.githubusercontent.com/Tomoku-dm/blog-images/master/8-fd.jpeg)



## 彩蛋：df 是怎么计算出来的

df Result = firstblock + superblocks + gdblocks + (2 + inode_blocks_per_group) * groups + journal_length

```
tune2fs -l /dev/vdb1| grep "Block count:"
Block count:              524288

tune2fs -l /dev/vdb1 | grep "First block:"
First block:              0

dumpe2fs /dev/vdb1 | grep superblock | wc -l
dumpe2fs 1.42.9 (28-Dec-2013)
6

dumpe2fs /dev/vdb1 | grep "Group descriptors" | wc -l
dumpe2fs 1.42.9 (28-Dec-2013)
6

dumpe2fs /dev/vdb1| grep ^Group | wc -l
dumpe2fs 1.42.9 (28-Dec-2013)
17
tune2fs -l /dev/vdb1 | grep "Inode blocks per group:"
Inode blocks per group:   512

dumpe2fs /dev/vdb1 | grep "Journal length"
dumpe2fs 1.42.9 (28-Dec-2013)
Journal length:           16384

0+6+6+（2+512）*17+16384=25134
25134是这些东西占的块数 *4096byte/1024=100536KB  和df 差的差不多
```


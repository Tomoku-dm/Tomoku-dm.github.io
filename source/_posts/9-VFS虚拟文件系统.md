---
title: VFS 虚拟文件系统
date: 2018-09-10 00:42:20
tags:
- kernel
- filesystem
---
### 前言
虚拟文件系统作为内核的子系统为用户空间程序和真正的多个文件系统直接提供了一个接口。简单来说。系统调用先和 VFS 交互，然后 VFS 提供的通用接口在被各个文件系统来实现。

**通用文件系统接口**  
虚拟文件系统，一个胶水层，位于内核的底层和用户层之间。它提供了各种抽象数据结构来表示文件和inode，而真实文件系统的实现必须填充这些结构，使得应用程序无需考虑底层文件系统，总是可以使用同样的接口访问和操作文件。在 VFS 和 内核 看来，底层的文件系统都是相同的，他们都支持文件、目录、Inode等概念。

![9-VFS](https://raw.githubusercontent.com/Tomoku-dm/blog-images/master/9-VFS.png)

> 进程调用系统调用 write(),系统调用 调用 VFS 中的通用接口 sys_wirte(),通用接口被底层真正的文件系统一一实现。

### UNIX 文件系统
VFS 文件系统的结构不在磁盘中真正存在，在系统启动时装载到内存中，VFS 结构与 EXT 文件系统相似，因为 EXT 文件系统是 linux 的原生文件系统。都有 file,directory,inode。

Unix 文件系统相关概念：
- file：文件对象主要是对进程而言，对应的操作函数基本和常见的系统调用一样。
- Inode：一个 Inode 对应一个文件，系统查找文件其实找的是 Inode，找到 Inode 后，通过对应关系去 data block 找到真正数据，Inode 中除了数据本身，文件的其他所有信息都包含，例如权限，访问时间，link等。
- surperblock: 文件系统的元数据，在mount 时，从 disk 的 superblock 读取到内存的 surperblock 对象中。
- dentry cache: 系统通过 Inode 在磁盘中找到文件的过程需要耗费大量时间，所以 VFS 引入了 dentry 对象来存放路径缓存，例如/mnt/dir/test.txt，其中/ 、mnt、dir、text.txt 都是一个 dentry 对象。有了缓存后，系统只需要在磁盘中找到 test.txt 的 inode 位置。 
- vfsmount，file_system_type  挂载点以及文件系统属性。
- fs_struct 和 file 偏向于进程相关的对象。



### VFS 的对象和数据结构

#### Superblock 

```c
//from <linux/fs.h>

// 在挂载时读入初始化
struct super_block {
    struct list_head            s_list;           /* list of all superblocks */
    ...
    struct super_operations     s_op;             /* superblock methods */
    struct list_head            s_inodes;           
    

struct super_operations {
struct inode *(*alloc_inode)(struct super_block *sb); void (*destroy_inode)(struct inode *);
void (*dirty_inode) (struct inode *);
int (*write_inode) (struct inode *, int);
void (*drop_inode) (struct inode *);
void (*delete_inode) (struct inode *);
void (*put_super) (struct super_block *);
void (*write_super) (struct super_block *);
int (*sync_fs)(struct super_block *sb, int wait); int (*freeze_fs) (struct super_block *);
int (*unfreeze_fs) (struct super_block *);
int (*statfs) (struct dentry *, struct kstatfs *);
int (*remount_fs) (struct super_block *, int *, char *);
void (*clear_inode) (struct inode *);
void (*umount_begin) (struct super_block *);
int (*show_options)(struct seq_file *, struct vfsmount *);
int (*show_stats)(struct seq_file *, struct vfsmount *);
...
}

sb->s_op->write_super(sb);
// C 不是面向对象的，需要这种方式来调用函数。但其实 VFS 中大多数概念都是面向对象的思想来实现的。

```
#### Inode
- 文件系统不管实际是否有 Inode ，想在 linux 中使用，必须将 Inode 虚拟化出来装载到内存中。
- 有的文件系统 没有 i_atime  i_mtime,  i_atime 概念，可以直接填写为 NULL，或者不往磁盘里刷脏。
- 可以看到 inode 的一些函数中传参都是 dentry，意味着在获取 Inode 之前，系统已经拿到对应的 dentry，也就是先有 dentry，再去找 inode。
```
//from <linux/fs.h>

struct inode_operations {
int (*create) (struct inode *,struct dentry *,int, struct nameidata *);
struct dentry * (*lookup) (struct inode *,struct dentry *, struct nameidata *); int (*link) (struct dentry *,struct inode *,struct dentry *);
int (*unlink) (struct inode *,struct dentry *);
int (*symlink) (struct inode *,struct dentry *,const char *);
int link(struct dentry *old_dentry, struct inode *dir,struct dentry *dentry)
//Invoked by the link() system call to create a hard link of the file old_dentry in the directory dir with the new filename dentry.
...
}
```

#### dentry
- dentry：目录项是描述文件的逻辑属性，只存在于内存中，并没有实际对应的磁盘上的描述，更确切的说是存在于内存的目录项缓存，为了提高查找性能而设计。注意不管是文件夹还是最终的文件，都是属于目录项，所有的目录项在一起构成一颗庞大的目录树。
- 一个有效的dentry结构必定有一个inode结构，这是因为一个目录项要么代表着一个文件，要么代表着一个目录，而目录实际上也是文件。所以，只要dentry结构是有效的，则其指针d_inode必定指向一个inode结构。
- dentry 有三个状态：
    - used：正在被用的，通过d_count记录用户数量，不能被回收。
    - unused：没有被用的，d_count是0，对应 Inode 是有效的。可以被回收
    - negative：对应 Inode 无效，被删除了，可以被回收。及时无效也不会立即被删除，因为同样有用，告知系统文件不存在。
- dentry cache 的存放：
    - used 通过i_dentry属性放在链表中。
    - 还有一个双向链表存放最近使用的dentry(unused and negative),链表头尾决定新旧程度，以及回收顺序。
    - hash 表`dentry_hashtable` 和 hash 函数`d_lookup()`，来定位要查找的 dentry。具体实现很复杂，详细请看相关文档。

```c
//from <linux/fs.h>

struct dentry { 
atomic_t        d_count; 
unsigned int    d_flags;
spinlock_t      d_lock;
int             d_mounted;
struct inode    *d_inode;           /* associated inode */ 
...
}

struct dentry_operations {
int (*d_revalidate) (struct dentry *, struct nameidata *);
int (*d_hash) (struct dentry *, struct qstr *);
int (*d_compare) (struct dentry *, struct qstr *, struct qstr *); int (*d_delete) (struct dentry *);
void (*d_release) (struct dentry *);
void (*d_iput) (struct dentry *, struct inode *);
char *(*d_dname) (struct dentry *, char *, int);
};

```

#### file
- 文件对象对应进程打开的文件，不是磁盘中的文件。
- 通过 `open()`系统调用创建，`close()`删除。
- 文件对象函数是我们那些常见的系统调用的基础，read（）write（）其实调用的就是 file 的函数。
```

struct file_operations { struct module *owner;
ssize_t (*read) (struct file *, char __user *, size_t, loff_t *); ssize_t (*write) (struct file *, const char __user *, size_t, loff_t *); ssize_t (*aio_read) (struct kiocb *, const struct iovec *,unsigned long, loff_t);
ssize_t (*aio_write) (struct kiocb *, const struct iovec *,unsigned long, loff_t);
int (*readdir) (struct file *, void *, filldir_t);
long (*unlocked_ioctl) (struct file *, unsigned int, unsigned long); long (*compat_ioctl) (struct file *, unsigned int, unsigned long); int (*mmap) (struct file *, struct vm_area_struct *);
int (*open) (struct inode *, struct file *);
int (*flush) (struct file *, fl_owner_t id);
int (*release) (struct inode *, struct file *);
int (*fsync) (struct file *, struct dentry *, int datasync);
int (*aio_fsync) (struct kiocb *, int datasync);
int (*fasync) (int, struct file *, int);
int (*lock) (struct file *, int, struct file_lock *);
```

#### files_struct
- 这是一个进程相关的对象，进程打开的文件和文件描述符相关的信息都在里面。
- fd_array 指向打开文件的链表。
```
//from <linux/fdtable.h>.

struct files_struct {
  /*
   * read mostly part
   */
        atomic_t count;
        struct fdtable __rcu *fdt;
        struct fdtable fdtab;
  /*
   * written part on a separate cache line in SMP
   */
        spinlock_t file_lock ____cacheline_aligned_in_smp;
        int next_fd;
        unsigned long close_on_exec_init[1];
        unsigned long open_fds_init[1];
        struct file __rcu * fd_array[NR_OPEN_DEFAULT];
};

```

> 进程如何找到打开的文件，通过文件描述符，file_struct,dentry,inode。
![8-fd](https://raw.githubusercontent.com/Tomoku-dm/blog-images/master/8-fd.jpeg)


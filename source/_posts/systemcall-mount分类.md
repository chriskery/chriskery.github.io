---
title: linux mount用法
date: 2023-11-28 22:22:38
tags:
- linux
- mount
categories:
- linux
- filesystem
---
## mount --bind
我们可以通过mount --bind命令来将两个目录连接起来，mount --bind命令是将前一个目录挂载到后一个目录上，所有对后一个目录的访问其实都是对前一个目录的访问，如下所示：
```
## test1 test2为两个不同的目录
linux-UMLhEm:/home/test/linux # ls test1
11.test  1.test
linux-UMLhEm:/home/test/linux # ls test2
22.test  2.test
linux-UMLhEm:/home/test/linux # ls -lid test1
1441802 drwx------ 2 root root 4096 Feb 13 09:50 test1
linux-UMLhEm:/home/test/linux # ls -lid test2
1441803 drwx------ 2 root root 4096 Feb 13 09:51 test2

## 执行mount --bind 将test1挂载到test2上，inode号都变为test1的inode
linux-UMLhEm:/home/test/linux # mount --bind test1 test2
linux-UMLhEm:/home/test/linux # ls -lid test1
1441802 drwx------ 2 root root 4096 Feb 13 09:50 test1
linux-UMLhEm:/home/test/linux # ls -lid test2
1441802 drwx------ 2 root root 4096 Feb 13 09:50 test2
linux-UMLhEm:/home/test/linux # ls test2
11.test  1.test

## 对test2的访问或修改实际上是改动test1目录
linux-UMLhEm:/home/test/linux # cd test2
linux-UMLhEm:/home/test/linux/test2 # touch 3.test
linux-UMLhEm:/home/test/linux/test2 # ls
11.test  1.test  3.test
linux-UMLhEm:/home/test/linux/test2 # cd ..
linux-UMLhEm:/home/test/linux # ls test1
11.test  1.test  3.test

## 解挂载后，test1目录保持修改，test2保持不变
linux-UMLhEm:/home/test/linux # umount test2
linux-UMLhEm:/home/test/linux # ls test1
11.test  1.test  3.test
linux-UMLhEm:/home/test/linux # ls test2
22.test  2.test

## 将test2挂载到test1上
linux-UMLhEm:/home/test/linux # ls -lid test2
1441803 drwx------ 2 root root 4096 Feb 13 09:51 test2
linux-UMLhEm:/home/test/linux # mount --bind test2 test1
linux-UMLhEm:/home/test/linux # ls -lid test1
1441803 drwx------ 2 root root 4096 Feb 13 09:51 test1
linux-UMLhEm:/home/test/linux # ls -lid test2
1441803 drwx------ 2 root root 4096 Feb 13 09:51 test2
linux-UMLhEm:/home/test/linux # ls test1
22.test  2.test
```

以mount --bind test1 test2为例，当mount --bind命令执行后，Linux将会把被挂载目录的目录项（也就是该目录文件的block，记录了下级目录的信息）屏蔽，即test2的下级路径被隐藏起来了（注意，只是隐藏不是删除，数据都没有改变，只是访问不到了）。同时，内核将挂载目录（test1）的目录项记录在内存里的一个s\_root对象里，在mount命令执行时，VFS会创建一个vfsmount对象，这个对象里包含了整个文件系统所有的mount信息，其中也会包括本次mount中的信息，这个对象是一个HASH值对应表（HASH值通过对路径字符串的计算得来），表里就有 /test1 到 /test2 两个目录的HASH值对应关系。

命令执行完后，当访问 /test2下的文件时，系统会告知 /test2 的目录项被屏蔽掉了，自动转到内存里找VFS，通过vfsmount了解到 /test2 和 /test1 的对应关系，从而读取到 /test1 的inode，这样在 /test2 下读到的全是 /test1 目录下的文件。

>- .mount --bind连接的两个目录的inode号码并不一样，只是被挂载目录的block被屏蔽掉，inode被重定向到挂载目录的inode（被挂载目录的inode和block依然没变）。
> - .两个目录的对应关系存在于内存里，一旦重启挂载关系就不存在了。

在固件开发过程中常常遇到这样的情况：为测试某个新功能，必需修改某个系统文件。而这个文件在只读文件系统上（总不能为一个小小的测试就重刷固件吧），或者是虽然文件可写，但是自己对这个改动没有把握，不愿意直接修改。这时候mount --bind就是你的好帮手。   
假设我们要改的文件是/etc/hosts，可按下面的步骤操作：   
1\. 把新的hosts文件放在/tmp下。当然也可放在硬盘或U盘上。   
2\. mount --bind /tmp/hosts /etc/hosts       此时的/etc目录是可写的，所做修改不会应用到原来的/etc目录，可以放心测试。测试完成了执行 umount /etc/hosts 断开绑定。

通过执行下面的命令可以创建一个挂载点：
```
mount --bind foo foo
```
在挂载后可以通过mount命令查看所有的挂载点.这样就可以使用一些mount的属性，最简单的例子，例如：
>sudo mount,ro --bind test_dir test_dir

可以让test_dir成为一个是read only的目录。无论改目录中的文件夹或者文件的权限是什么，这个文件夹都是只读的。

如果要递归的挂载一个目录可以使用如下命令.
mount --rbind olddir newdir
递归的挂载是指如果挂载的olddir内有挂载点，会把这个挂载点也一起挂载到newdir下。
## flags
```
/* 
 * These are the fs-independent mount-flags: up to 32 flags are supported 
 */  
#define MS_RDONLY        1         /* 对应-o ro/rw */  
#define MS_NOSUID        2         /* 对应-o suid/nosuid */  
#define MS_NODEV         4         /* 对应-o dev/nodev */  
#define MS_NOEXEC        8         /* 对应-o exec/noexec */  
#define MS_SYNCHRONOUS  16         /* 对应-o sync/async */  
#define MS_REMOUNT      32         /* 对应-o remount，告诉mount这是一次remount操作 */  
#define MS_MANDLOCK     64         /* 对应-o mand/nomand */  
#define MS_DIRSYNC      128        /* 对应-o dirsync */  
#define MS_NOATIME      1024       /* 对应-o atime/noatime */  
#define MS_NODIRATIME   2048       /* 对应-o diratime/nodiratime */  
#define MS_BIND         4096       /* 对应-B/--bind选项，告诉mount这是一次bind操作 */  
#define MS_MOVE         8192       /* 对应-M/--move，告诉mount这是一次move操作 */  
#define MS_REC          16384      /* rec是recursive的意思，这个flag一般不单独出现，都是伴随这其它flag，表示递归的进行操作 */  
#define MS_VERBOSE      32768      /* 对应-v/--verbose */  
#define MS_SILENT       32768      /* 对应-o silent/loud */  
#define MS_POSIXACL     (1<<16)    /* 让VFS不应用umask，如NFS */  
#define MS_UNBINDABLE   (1<<17)    /* 对应--make-unbindable */  
#define MS_PRIVATE      (1<<18)    /* 对应--make-private */  
#define MS_SLAVE        (1<<19)    /* 对应--make-slave */  
#define MS_SHARED       (1<<20)    /* 对应--make-shared */  
#define MS_RELATIME     (1<<21)    /* 对应-o relatime/norelatime */  
#define MS_KERNMOUNT    (1<<22)    /* 这个一般不在应用层使用，一般内核挂载的文件系统如sysfs使用，表示使用kern_mount()进行挂载 */  
#define MS_I_VERSION    (1<<23)    /* 对应-o iversion/noiversion */  
#define MS_STRICTATIME  (1<<24)    /* 对应-o strictatime/nostrictatime */  
#define MS_LAZYTIME     (1<<25)    /* 对应 -o lazytime/nolazytime*/

/* 下面这几个flags都是内核内部使用的，不由mount系统调用传递 */
#define MS_SUBMOUNT     (1<<26)
#define MS_NOREMOTELOCK (1<<27)
#define MS_NOSEC        (1<<28)
#define MS_BORN         (1<<29)
#define MS_ACTIVE       (1<<30)
#define MS_NOUSER       (1<<31)
/* 
 * Superblock flags that can be altered by MS_REMOUNT 
 */  
#define MS_RMT_MASK     (MS_RDONLY|MS_SYNCHRONOUS|MS_MANDLOCK|MS_I_VERSION|\                   
                         MS_LAZYTIME)  // 可以在remount时改变的flags  
  
/* 
 * Old magic mount flag and mask 
 */  
#define MS_MGC_VAL 0xC0ED0000      /* magic number */  
#define MS_MGC_MSK 0xffff0000      /* flags mask */
```
除了上面这些flags对应的mount选项，剩下的基本就是data来传递。

## mount/umount函数详解
```
功能描述：
mount挂上文件系统，umount执行相反的操作。
  
用法：  
#include <sys/mount.h>

int mount(const char *source, const char *target,
   const char *filesystemtype, unsigned long mountflags, const void *data);

int umount(const char *target);

int umount2(const char *target, int flags);
```
参数
```
source：将要挂上的文件系统，通常是一个设备名。
target：文件系统所要挂在的目标目录。
filesystemtype：文件系统的类型，可以是"ext2"，"msdos"，"proc"，"nfs"，"iso9660" 。。。
mountflags：指定文件系统的读写访问标志，可能值有以下

MS_BIND：执行bind挂载，使文件或者子目录树在文件系统内的另一个点上可视。
MS_DIRSYNC：同步目录的更新。
MS_MANDLOCK：允许在文件上执行强制锁。
MS_MOVE：移动子目录树。
MS_NOATIME：不要更新文件上的访问时间。
MS_NODEV：不允许访问设备文件。
MS_NODIRATIME：不允许更新目录上的访问时间。
MS_NOEXEC：不允许在挂上的文件系统上执行程序。
MS_NOSUID：执行程序时，不遵照set-user-ID 和 set-group-ID位。
MS_RDONLY：指定文件系统为只读。
MS_REMOUNT：重新加载文件系统。这允许你改变现存文件系统的mountflag和数据，而无需使用先卸载，再挂上文件系统的方式。
MS_SYNCHRONOUS：同步文件的更新。
MNT_FORCE：强制卸载，即使文件系统处于忙状态。
MNT_EXPIRE：将挂载点标志为过时
```

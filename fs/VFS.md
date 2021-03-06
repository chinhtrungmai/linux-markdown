Virtual File System
========================================

VFS不仅为文件系统提供了方法和抽象，还支持文件系统中对象（或文件）的统一视图。
文件这个术语的含义看起来似乎很清楚，但由于各个文件系统的底层实现不同，其语义
经常有许多小而微妙的差异。并非所有文件系统都支持同样的功能，而有些操作（对
“普通”文件是不可缺少的）对某些对象完全没有意义，例如集成到VFS中的命名管道。

并非每一种文件系统都支持VFS中的所有抽象。设备文件无法存储在源自其他系统的
文件系统中（如FAT），后者的设计没有考虑到此类对象。

定义一个最小的通用模型，来支持内核中所有文件系统都实现的那些功能，这是不实际的。
因为这样会损失许多本质性的功能特性，或者导致这些特性只能通过特定文件系统的路径访问。
这实际上否定了虚拟抽象层所带来的好处。

VFS的方案完全相反：提供一种结构模型，包含了一个强大文件系统所应具备的所有组件。
但该模型只存在于虚拟中，必须使用各种对象和函数指针与每种文件系统适配。所有文件
系统的实现都必须提供与VFS定义的结构配合的例程，以弥合两种视图之间的差异。当然，
虚拟文件系统的结构并非是幻想出来的东西，而是基于描述经典文件系统所使用的结构。
VFS抽象层的组织显然也与Ext2文件系统类似。这对于基于完全不同概念的文件系统来说，
会更加困难（例如，Reiser或XFS文件系统），但处理Ext2文件系统时会提高性能，
因为在Ext2和VFS结构之间转换，几乎不会损失时间。

在处理文件时，内核空间和用户空间使用的主要对象是不同的。对用户程序来说，一个文件由一个
文件描述符标识。该描述符是一个整数，在所有有关文件的操作中用作标识文件的参数。文件描述符
是在打开文件时由内核分配，只在一个进程内部有效。两个不同进程可以使用同样的文件描述符，
但二者并不指向同一个文件。基于同一个描述符来共享文件是不可能的。内核处理文件的关键是inode。
每个文件（和目录）都有且只有一个对应的indoe，其中包含元数据（如访问权限、上次修改的日期，等等）
和指向文件数据的指针。但inode并不包含一个重要的信息项，即文件名，这看起来似乎有些古怪。
通常，假定文件名称是其主要特征之一，因此应该被归入用于管理文件的对象（inode）中。

VFS Framework
----------------------------------------

在VFS接口的实现中，涉及大量数据结构，有些非常冗长。VFS由两个部分组成：文件和文件系统，
这些都需要管理和抽象。

![vfs_frame.jpg](https://github.com/novelinux/linux-4.x.y/tree/master/fs/res/vfs_frame.jpg)

在抽象对底层文件系统的访问时，并未使用固定的函数，而是使用了函数指针。这些函数指针保存在
两个结构中，包括了所有相关的函数。

* inode操作:

创建链接、文件重命名、在目录中生成新文件、删除文件。

* 文件操作:

作用于文件的数据内容。它们包含一些显然的操作（如读和写），还包括如设置文件位置
指针和创建内存映射之类的操作。

除此之外，还需要其他结构来保存与inode相关的信息。特别重要的是与每个inode关联的数据段，
其中存储了文件的内容或目录项表。每个inode还包含了一个指向底层文件系统的超级块对象的指针，
用于执行对inode本身的操作（这些操作也是通过函数指针数组实现）。还可以提供有关文件系统
特性和限制的信息。

因为打开的文件总是分配到系统中一个特定的进程，内核必须在数据结构中存储文件和进程之间的关联。
task_struct包含一个成员，其中保存了所有打开的文件（通过一种迂回方式）。该成员是一个数组，
访问时使用文件描述符作为索引。各个数组项包含的对象不仅关联到对应文件的inode，还包含一个指针，
指向用于加速查找操作的目录项缓存的一个成员。各个文件系统的实现也能在VFS inode中存储自身的
数据(不通过VFS层操作)。

Data Structure
----------------------------------------

### struct file_system_type

https://github.com/novelinux/linux-4.x.y/tree/master/include/linux/fs.h/struct_file_system_type.md

### struct vfsmount

每个装载的文件系统都对应于一个vfsmount结构的实例，其定义如下：

https://github.com/novelinux/linux-4.x.y/tree/master/include/linux/mount.h/struct_vfsmount.md

### struct super_block

https://github.com/novelinux/linux-4.x.y/tree/master/include/linux/fs.h/struct_super_block.md

### struct inode

https://github.com/novelinux/linux-4.x.y/tree/master/include/linux/fs.h/struct_inode.md

### struct iattr

https://github.com/novelinux/linux-4.x.y/tree/master/include/linux/fs.h/struct_iattr.md

### struct file

https://github.com/novelinux/linux-4.x.y/tree/master/include/linux/fs.h/struct_file.md

### struct fd

https://github.com/novelinux/linux-4.x.y/tree/master/include/linux/file.h/struct_fd.md

### struct address_space

https://github.com/novelinux/linux-4.x.y/tree/master/include/linux/fs.h/struct_address_space.md

### struct task_struct

文件描述符（就是整数）用于在一个进程内唯一地标识打开的文件。这假定了内核能够在用户进程中
的描述符和内核内部使用的结构之间，建立一种关联。每个进程的task_struct中包含了用于完成该
工作的成员。

```
/* filesystem information */
    struct fs_struct *fs;
/* open file information */
    struct files_struct *files;
/* namespaces */
    struct nsproxy *nsproxy;
```

https://github.com/novelinux/linux-4.x.y/tree/master/include/linux/sched.h/struct_task_struct.md

#### struct fs_struct

https://github.com/novelinux/linux-4.x.y/tree/master/include/linux/fs_struct.h/struct_fs_struct.md

#### struct files_struct

https://github.com/novelinux/linux-4.x.y/tree/master/include/linux/fdtable.h/struct_files_struct.md

#### struct nsproxy

https://github.com/novelinux/linux-4.x.y/tree/master/include/linux/nsproxy.h/README.md

register_filesystem
----------------------------------------

https://github.com/novelinux/linux-4.x.y/blob/master/fs/filesystems.c/register_filesystem.md

Initialization
----------------------------------------

https://github.com/novelinux/linux-4.x.y/tree/master/init/fs/README.md

APIS
----------------------------------------

### About File System

#### mount

目录树的装载和卸载比仅仅注册文件系统复杂得多，因为后者只需要向一个链表添加对象，而前者需要对
内核的内部数据结构执行很多操作，所以要复杂得多。文件系统的装载由mount系统调用发起。

#### umount

### About File

#### open

https://github.com/novelinux/system_calls/blob/master/open.md

#### write

https://github.com/novelinux/system_calls/blob/master/write.md

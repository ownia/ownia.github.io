---
title: What We Talk About When We Talk About ChrDev
date: 2020-09-17
---

> 本文内核版本主要基于4.14，少部分会与5.8.7版本进行比较。

## 什么是设备驱动程序

Linux内核拥有庞大而复杂的代码，通常内核黑客(kernel hacker)需要找到一个进入内核的方法，设备驱动程序就是其中一个选择。设备驱动程序作为应用软件和硬件之间的纽带，软件上层调用API接口，通过中间的设备驱动程序驱使底层硬件设备进行具体工作。设备驱动程序通常不会提供具体的策略，而是处理硬件适用的问题，用以展现设备的特性。

## 设备驱动包括哪些

作为不太严格的划分，Linux系统设备通常分为三类：字符设备、块设备和网络设备。字符设备和块设备的区别在于内核传递数据的方式不一致，两者都是通过文件系统节点来访问。当然，Linux还有其他类型的设备用以实现其他的任务，但总的来看，内核可以通过动态的加载内核模块实现具体的功能。

使用`ll /dev`查看设备文件的具体信息：

![](/images/what_we_talk_about_when_we_talk_about_chrdev/20200917112623.png)


可以看到，文件属性字段的第一个字母直接表明了该设备的种类，字母c代表字符设备文件，字母b代表块设备文件。同时，原本标识文件长度的地方现在变为了两个编号，这是因为Linux内核采用主设备号和次设备号标识一个确定的设备驱动程序。一个驱动程序可以分配多个主设备号，同一个设备驱动程序可以管理多个同样类型的设备，通过次设备号来标识。

下文主要讨论字符设备。

## 上层应用与底层驱动

在Linux中，由于“一切皆文件”的性质，对于硬件设备的操作在应用中就被视作了文件的操作，每个文件被一个struct inode结构体来描述，而对于字符设备驱动的inode来说，其中包含一个struct cdev结构体记录字符设备。当open函数打开设备文件时，根据inode结构中的设备信息可以判断该设备文件是字符设备还是块设备并分配一个struct file，之后根据inode中dev_t类型的设备号找到对应的驱动程序。同时将struct cdev的内存空间首地址存储到inode中的i_cdev成员中，将struct cdev的接口函数操作地址存储到file结构体的f_op成员中。最终VFS层会给open函数返回一个文件描述符fd(与struct file对应)，其他函数就可以根据这个文件描述符fd找到struct file，并进行对应字符设备的函数操作。

![](/images/what_we_talk_about_when_we_talk_about_chrdev/chrdev12.jpg)

其中在VFS层中的操作过程会在下面分析。

## 设备号及分配

对于字符设备来说有两种方法分配设备号：mknod和使用函数在驱动程序中创建。而在使用函数在驱动程序中创建时，通常有两个选择，静态分配(`register_chrdev_region()`)和动态分配(`alloc_chrdev_region()`)。在分配成功时返回0，失败时返回负的错误码。还可以使用`include/linux/kdev_t.h`中特定的宏来进行设备号相关操作，`MAJOR(dev_t dev)`可以获取主设备号，`MINOR(dev_t dev)`可以获取次设备号，`MKDEV(int major, int minor)`可以将一对主次设备号转换为dev_t类型的设备编号。

对于内核开发者来说，不能向自己的设备驱动程序随便定义设备号。对于内核源码的标准发行版，在`Documentation/admin-guide/devices.txt`下可以看到常用设备分配的设备号，在进行静态分配设备号时需要避开防止冲突。

动态注册设备号允许在用户层来创建，当内核检测到设备时，通过udevd守护进程机制，借助udevd规则创建内核对象进行设备管理，借助tmpfs在/dev中创建对应项，由于tmpfs的特性，设备节点不再是持久的，在系统关机重启后会消失。

`register_chrdev_region()`和`alloc_chrdev_region()`函数都在`fs/char_dev.c`中定义。静态分配时，`register_chrdev_region()`通过传入的dev_t设备编号和设备数确定注册的范围，并在这个范围中遍历注册字符设备，注册失败则调用`__unregister_chrdev_region()`来卸载字符设备并使用kfree来释放kzalloc申请的空间。对于`__unregister_chrdev_region()`来说，它多了一个baseminor参数来确定要求注册的次设备号范围的第一个序号，同时在内部调用`__register_chrdev_region()`时所传入的major值为0，这样`__register_chrdev_region()`判断major等于0时，通过`find_dynamic_major()`函数从散列表的末尾表项开始继续向后寻找一个与尚未使用的主设备号对应的空冲突表。

如果观察`register_chrdev_region()`和`alloc_chrdev_region()`的异常判断和返回值，可以发现它们都使用了两个宏：`IS_ERR()`和`PTR_ERR()`，这是两个很有趣的实现。

在`include/linux/err.h`中实现了一个宏`#define IS_ERR_VALUE(x) unlikely((unsigned long)(void *)(x) >= (unsigned long)-MAX_ERRNO)`来判断是否是错误码，并且这个宏对不同的体系架构具有普适性(a per-architecture thing)。Linux内核中，错误码都是以负数的形式存在，由`#define MAX_ERRNO 4095`可知，错误码的范围为[-4095,-1]。在unsigned long下面，-4095转换为0xFFFFF001，-1转换为0xFFFFFFFF，所以上述范围就变为了[0xFFFFF001,0xFFFFFFFF]。

Linux上用户空间和内核空间是被分开的，在内核空间中操作系统划分出一块保留区域，这个范围内的内核地址是不能被分配的。

![](/images/what_we_talk_about_when_we_talk_about_chrdev/chrdev13.jpg)

由上图可知这块保留区域在内存的中的地址恰好就是之前unsigned long转换后错误码的范围。`IS_ERR()`传入的是一个指针类型的参数，它直接判断这个传递的指针是否是在有效的内存地址，此时如果传入的是转换为指针类型的错误码就会判断为保留地址，即得出这个传入的指针不是分配的内存地址、而是被转换为指针类型的错误码的结论。之后`PTR_ERR()`直接将这个指针转换为long类型的错误码值并返回。

通过利用保留区域和MAX_ERRNO，成功将有效的内存地址和错误码区分开来。

## 字符设备

字符设备作为设备文件，通过inode中的成员i_cdev直接指向cdev结构体。在`include/linux/cdev.h`中可以看见cdev的原型。
```
struct cdev {
	struct kobject kobj;
	struct module *owner;
	const struct file_operations *ops;
	struct list_head list;
	dev_t dev;
	unsigned int count;
} __randomize_layout;
```
kobject是内核设备模型的核心部分，提供了引用计数、父指针等字段。当初始化之后，kobject的引用计数设为1，如果引用计数不清零，则该对象就一直保持在内存中。当引用计数为零时，对象可以被撤销，相关内存也可以被释放。通过`kref_get()`和`kref_put()`可以增加和减少引用计数。

当kobject嵌入cdev中时，cdev结构体成为对象模型层次架构中的一部分，使得cdev能拥有`cdev->kobj.parent`等指针操作。owner表示驱动程序模块所有者字段。list作为一个链表来包含设备特殊文件的inode。dev是dev_t类型的设备号。count用来记录该设备下面关联的次设备号的数量。ops作为最重要的的成员，是一个file_operations结构体，内核通过`cdev->ops`直接访问该设备的文件操作。

file_operations结构包含了大量的函数指针，对于每一个所要设置的方法，对应的每个字段都应该指向驱动程序中实现对应操作的函数，而对于不需要实现的方法可以将其设置为NULL。在早期，字符设备的文件操作定义很含糊，在标准上只有一个open操作来向结构传入已经打开的设备的函数指针，从而来进一步操作字符设备。如今字符设备驱动的文件操作越来越多，包含了各种对于设备属性的抽象。除开我们在设备驱动程序中自己编写的文件操作，其实内核本身也提供了预留的文件操作，它们通常以`generic_file_`作为函数名的开头，例如`generic_file_open()`、`generic_file_read_iter()`、`generic_file_llseek()`等，这些通用实现为文件操作提供了便利。

由于现实中字符设备驱动的复杂性和多样的设计需求，struct cdev通常是作为一个内嵌的成员变量放在实际的字符设备的数据结构中的。

在对设备节点分配完成后，在内核对设备进行操作之前，由于内核使用struct cdev结构来表示字符设备，因此需要进行cdev的分配和初始化。由于我们将cdev嵌入到自己的数据结构中，所以通过`cdev_init()`初始化已经分配到的结构并绑定已经定义的file_operations。之后调用`cdev_add()`向内核添加该字符设备，同样，它也是成功返回0，失败返回负的错误码。当需要移除字符设备时，通过`cdev_del()`进行移除。

那么对于一个最基本的read文件操作，是怎么通过用户空间的C库API传递到底层文件设备进行读取的呢？让我们先从VFS入手。

## VFS

VFS全称Virtual File System，是内核子系统，为用户空间提供文件和文件系统相关的接口。要理解虚拟文件系统，应先看看Linux下的文件系统。VFS的主要代码位于`include\linux\fs.h`。Linux秉承“一切皆文件”的理念，整个系统由成千上万的文件构成，使用目录结构来管理组织存储的数据。Linux支持不同的文件系统，如Ext2、Ext4、XFS、Btrfs、VFAT等，然而不同的文件系统具有非常大的特性差异，为了将这些具体特性与上层应用层和下层内核层分离开来，需要提供一层关键的文件处理机制，这就是VFS。VFS向下对底层独立的文件系统导出接口进行抽象，向上提供用户空间到系统调用访问文件系统的功能。

![](/images/what_we_talk_about_when_we_talk_about_chrdev/chrdev11.jpg)

对于每个文件系统的实现来说，它们导出一组通用接口提供给VFS。对于块设备文件来说，使用页缓存、块缓存等缓冲区用来缓存文件系统与块设备层之间的请求。VFS采取一种大而全的思想，它提供了一种结构模型，包含了文件系统所具备的全部组件，然后通过函数指针来进行定义。

对于用户空间和内核空间来说，两者之间在处理文件时面向的对象是不一样的。对于用户程序来说，一个文件由一个文件描述符标识，文件描述符由内核分配，只在一个进程内部有效。对于内核来说，则是通过inode来描述文件的元数据和指向文件数据的指针。inode_hashtable支持根据inode编号和超级块快速访问inode。

我们先从4个主要的对象类型来分析VFS：super_operations(超级块对象)、inode_operations(索引节点对象)、dentry_operations(目录项对象)、file_operations(文件对象)。每种对象作为一个结构体指针来实现，每个结构体里面包含对应的指向其文件操作的函数指针。这4种对象由4个结构体来声明：super_block、inode、dentry、file。

各类文件系统都要实现超级块对象，它用来描述整个文件系统的信息，包括设备标识符、文件系统子类型、文件大小上限、超级块方法、挂载标志等。索引节点对象包括了内核操作文件及目录时的信息，如散列表、访问权限、引用计数、实际设备标识符、相关的超级块、相关的地址映射等。

VFS把目录当作文件来看待，所以目录项能用来表示具体文件的位置，并反映了文件系统的树状关系。目录项对象没有对应的硬盘对象结构，因此也不存在是否被修改的标志。目录项对象具有4种状态：空闲、被使用、未被使用、负状态。处于空闲状态的目录项对象不包括有效的信息，且还没有被VFS使用。被使用状态对应一个有效的inode，且d_count为正值。未被使用状态也对应一个有效的inode，但d_count为0，该目录项不会快速撤销，有利于之后继续使用。负状态的目录项没有对应的有效inode，虽然索引节点已经被删除，但是目录项还是被保留。dentry结构不仅易于处理文件系统，还能通过dcache提高系统性能。当访问一个路径时，如果能在目录项缓存中找到该路径名，就能节约大量时间，就算没有找到该路径名，也能将其解析之后放入缓存以便以后快速查找。由于文件系统的特性，文件访问通常是具有空间及时间的局部性，程序很有可能会在同一个目录下访问多个文件或者某个文件会被多次访问，此时由于dcache的存在，其路径命中几率会大大增加。

文件对象表示进程打开的文件在其内存中的表示，用于搭建进程和磁盘上的文件的对应关系。由于多个进程可能同时操作同一个文件，因此一个文件可能存在多个对应的文件对象，但这个文件对应的inode和dentry应该是唯一的。

除了4大文件对象外，还有其他很多与VFS相关的数据结构如vfsmount(表示文件系统的安装点)、file_system_type(描述文件系统功能结构)、fiemap_extent(基于extent存储的数据结构)等。

那么VFS的这些文件对象如何和进程关联到一起？主要是下面这3种数据结构：files_struct、fs_struct、namespace。文件描述符fd是用来描述打开的文件的，每个进程用一个files_struct结构来记录文件描述符的使用情况，这个files_struct结构称为用户打开文件表，它是进程的私有数据。fs_struct由进程描述符的fs指向，包括了文件系统与进程相关的信息，其中count作为共享同一fs_struct的进程数目，同时用dentry来表示根目录、当前目录等。namespace由进程描述符的mmt_namespace指向，通常来说内核使用的是单进程命名空间。

一个文件系统对应一个超级块和一个vfsmount。超级块中的s_type指向具体的file_system_type，具体的文件系统通过fs_supers链接具有同一种文件类型的超级块，同一种文件系统类型的超级块由s_instances进行链接。之后进程通过task_struct中的files找到files_struct结构，其中具有成员文件描述符数组fd_array，也就是文件对象数组的索引值。文件对象通过域f_dentry找到d_inode对应的索引节点，建立起文件对象与进程的关联。

## 从read出发

在了解VFS之后，我们回过头来看看用户空间的read到底是怎样读取字符设备的。

假如现在已经实现了一个简单的字符设备驱动，且通过模块的方式加载了驱动，并赋予设备`/dev/dev_fifo0`相应的权限。编写了一个测试文件如下：
```c
#include <stdio.h>
#include <unistd.h>
#include <fcntl.h>

int main() {
    int fd;
    char key_val[64] = {0};
    fd = open("/dev/dev_fifo0", O_RDWR);
    read(fd, &key_val, 64);
    close(fd);
    return 0;
}
```
该测试程序直接调用read系统函数，函数原型在`<unistd.h>`中定义如下
`ssize_t read (int __fd, void *__buf, size_t __nbytes)`
运行`$ man 2 read`了解函数相关信息
对应的三个参数为文件描述符、缓冲区起始地址、字节数。根据描述，在支持查找的文件上，读取操作从文件偏移量f_pos开始，以读取的字节数nbytes递增，若文件偏移量等于或大于文件结尾，则停止read。当读取完成所有字节数则返回0，如果返回一个比字节数小的正值则表示读取成功但可能遇到EOF、管道关闭或信号中断，如果读取错误则返回负的错误码。
由于使用系统函数，所有它通过系统调用执行陷入指令切换到内核态。为了追踪这个系统函数，我们需要对可执行文件进行反编译来分析汇编文件。

```
$ arm-linux-gnueabihf-gcc -static -g test.c -o test
$ arm-linux-gnueabihf-objdump -D -S test > test.s
```
查看汇编代码：

![](/images/what_we_talk_about_when_we_talk_about_chrdev/20200917095943.png)

在main中可以看到，当运行到`read(fd, &key_val, 64);`时，通过子程序跳转bl跳转到`__libc_read`。

![](/images/what_we_talk_about_when_we_talk_about_chrdev/20200917100022.png)

之后在`__libc_read`中，mov.w指令将立即数3传给16位的ip指令指针寄存器，在下一条指令通过bl跳转到`__libc_do_syscall`。

![](/images/what_we_talk_about_when_we_talk_about_chrdev/20200917100203.png)

在`__libc_do_syscall`中，mov指令将ip指令指针寄存器传给r7寄存器，然后在下一条指令中用svc系统调用指令转到内核空间中。此处的svc就是ARM汇编中swi的更新替代指令，在发生软中断后直接跳转到异常向量表处。

在`arch/arm/kernel/entry-common.S`里可以看到swi的入口`ENTRY(vector_swi)`。在这下面我们可以看到一段注释`Pure EABI user space always put syscall number into scno (r7).`

## OABI和EABI

ABI，全称application binary interface，应用程序二进制接口，它使得两个二进制程序模块能够互相兼容。OABI全称old application binary interface，EABI全称embedded application binary interface，两者都是基于ARM平台来讲的。OABI作为ARM系列的第一个ABI，它假设了一个具有浮点运算单元的ARM平台，然而这在没有FPU的机器上会一直请求通信引发内核异常。同时，浮点计算效率也不高。EABI解决了上述问题，优化了嵌入式系统中有限资源内的性能，其浮点性能比OABI高了10倍。

内核对于OABI和EABI给出了两个system call table，可在`arch/arm/include/generated/calls-oabi.S`和`arch/arm/include/generated/calls-eabi.S`找到它们。

通常来说，可以在make menuconfig中对OABI和EABI进行配置。如果想查看经过某个工具链编译之后的可执行文件采取哪一种配置，可以通过下面的指令进行判断：
`$ readelf -h [file]`

![](/images/what_we_talk_about_when_we_talk_about_chrdev/20200917095739.png)

可以看到，在经过arm-linux-gnueabihf-gcc工具链编译之后可执行文件的ABI配置为`Version5 EABI, hard-float ABI`。

## syscall table

现在回到`entry-common.S`。
通过`addne scno, r7,`，scno成为了寄存器r7的别名，其值为系统调用号3，`adr tbl, sys_call_table`读取系统调用表的地址给tbl。

之后先用对CONFIG_AEABI进行判断决定查找`calls-eabi.S`还是`calls-oabi.S`。
```
#ifdef CONFIG_AEABI
#include <calls-eabi.S>
#else
#include <calls-oabi.S>
#endif
```
最终通过`ldrlo pc, [tbl, scno, lsl #2]`来调用需求的系统调用。由于`__NR_OABI_SYSCALL_BASE`是0，所以scno中的值还是为系统调用号3，`lsl #2`左移两位乘4，tbl为系统调用表sys_call_table的基地址。`calls-eabi.S`中每个系统调用标号占用4个字节(NATIVE)，所以在基地址上加系统调用号3乘4的值，就可以直接跳入执行`sys_read`。

`sys_read`在哪呢？我们在`fs/read_write.c`中可以找到它的定义：`SYSCALL_DEFINE3(read, unsigned int, fd, char __user *, buf, size_t, count)`。

## 宏定义的艺术

SYSCALL_DEFINE3作为一个明显的宏，让我们追根溯源一下看看它到底如何展开。先从`include/linux/syscalls.h`看起。

```
#define SYSCALL_DEFINE3(name, ...) SYSCALL_DEFINEx(3, _##name, __VA_ARGS__)
```
其中省略号代表可变部分，双#进行分割链接，单#将参数转为字符串。此时宏被展开为`SYSCALL_DEFINEx(3, _read, __VA_ARGS__)`。
根据

```
#define SYSCALL_DEFINEx(x, sname, ...)				\
	SYSCALL_METADATA(sname, x, __VA_ARGS__)			\
	__SYSCALL_DEFINEx(x, sname, __VA_ARGS__)
```
此时宏被展开为
```
	SYSCALL_METADATA(_read, 3, __VA_ARGS__)			\
	__SYSCALL_DEFINEx(3, _read, __VA_ARGS__)
```
为了后面高效率的分析宏展开，我们假设此时CONFIG_FTRACE_SYSCALLS未定义。根据
```
#define __SYSCALL_DEFINEx(x, name, ...)					\
	asmlinkage long sys##name(__MAP(x,__SC_DECL,__VA_ARGS__))	\
...
```
可将之前的宏展开为
```
asmlinkage long sys_read(__MAP(3,__SC_DECL,__VA_ARGS__))
```
同时根据
```
#define __MAP0(m,...)
#define __MAP1(m,t,a) m(t,a)
#define __MAP2(m,t,a,...) m(t,a), __MAP1(m,__VA_ARGS__)
#define __MAP3(m,t,a,...) m(t,a), __MAP2(m,__VA_ARGS__)
#define __MAP4(m,t,a,...) m(t,a), __MAP3(m,__VA_ARGS__)
#define __MAP5(m,t,a,...) m(t,a), __MAP4(m,__VA_ARGS__)
#define __MAP6(m,t,a,...) m(t,a), __MAP5(m,__VA_ARGS__)
#define __MAP(n,...) __MAP##n(__VA_ARGS__)
#define __SC_DECL(t, a)	t a
```
最终将宏展开为
```
asmlinkage long sys_read(unsigned int fd, char __user *buf, size_t count);
```
这就是系统调用sys_read的原样。经过这些宏嵌套、宏展开，我们可以知道SYSCALL_DEFINE后面所带的数字就是参数的个数n，其内部第一个参数为系统调用名字的后半部分，之后紧跟n对系统调用参数类型及名称。

为什么要使用如此多的宏定义而不是采取直接展开的方式呢？主要是因为在以前旧版本的内核中，对于不同体系架构的平台来讲，在用户空间里要将系统调用的32位参数在64位寄存器中进行正确的符号扩展，但由于某些平台的原因导致这个扩展可能失败，访问到错误的地址空间或产生其他异常，导致系统崩溃。

于是内核增加了宏展开将所有参数转换为64位long类型，在去为所需的类型的、进行转换，避免有符号、无符号之间的错误。

宏的艺术不止于此，例如在`include/linux/build_bug.h`中有更精彩的实现。

## read之后

把视线切换回到`fs/read_write.c`，可以发现在sys_read中有一个重要的函数`vfs_read()`。在内核版本5.8.7中多了一层ksys_read。
`vfs_read()`原型如下：
`ssize_t vfs_read(struct file *file, char __user *buf, size_t count, loff_t *pos)`
sys_read先通过`file_pos_read()`读取file的f_pos，即文件读写位置，最终向`vfs_read()`传入file结构、用户空间缓存、读取字节长度、文件读取位置。在确认读写区域有效及读写长度在范围内之后，执行双下划线的底层调用：
```
ssize_t __vfs_read(struct file *file, char __user *buf, size_t count,
		   loff_t *pos)
{
	if (file->f_op->read)
		return file->f_op->read(file, buf, count, pos);
	else if (file->f_op->read_iter)
		return new_sync_read(file, buf, count, pos);
	else
		return -EINVAL;
}
```
结合之前在VFS的信息，可以知道，file直接找到成员f_op，即一个file_operations对象。通过f_op直接找到字符设备中的设备读取方法，实现字符设备驱动中的设备读写。读写成功后，函数`fsnotify_access()`就能通知文件被读取，`add_rchar()`增加当前进程读取字节数，然后通过`inc_syscr()`增加当前进程系统调用次数。

就经过这样一个流程，用户空间的read就与内核空间的字符设备read连结在一起。

## 系统调用与库函数

read与fread有什么不同？系统调用是面向底层的，直接通向操作系统内部的接口，用户态通过系统调用利用软中断切换到内核态，在内核中通过调用内核相关函数实现相应功能。而库函数通常是在用户空间地址执行的，通过将API进行封装和整合， 面向复杂情况下的应用开发。由于大部分库函数并不使用系统调用，不会产生内核上下文切换，因此库函数的效率远远大于系统调用。即使在某些库函数中使用到系统调用，但由于库函数采取缓冲区等措施，因此也比本身系统调用的性能高。

## GOT和PLT

当我们使用x86_64-linux-gcc动态编译时，与静态编译有什么不同？
```
$ gcc -g test.c
$ objdump -D -S a.out > objdump.s
```
查看汇编代码，可以在main里得到如下信息：

![](/images/what_we_talk_about_when_we_talk_about_chrdev/20200918102441.png)

程序跳转为地址为0x10a0的`read@plt`。

![](/images/what_we_talk_about_when_we_talk_about_chrdev/20200918102517.png)


可以看出，通过`4c0: e5bcfb00 ldr pc, [ip, #2816]!;`，将ip寄存器值加2816偏移量读入程序计数器，并将新地址ip+2816写入中断优先寄存器，进入中断。

使用`$ objdump -R a.out`得出
```
a.out:     file format elf64-x86-64

DYNAMIC RELOCATION RECORDS
OFFSET           TYPE              VALUE 
0000000000003da0 R_X86_64_RELATIVE  *ABS*+0x00000000000011a0
0000000000003da8 R_X86_64_RELATIVE  *ABS*+0x0000000000001160
0000000000004008 R_X86_64_RELATIVE  *ABS*+0x0000000000004008
0000000000003fd8 R_X86_64_GLOB_DAT  _ITM_deregisterTMCloneTable
0000000000003fe0 R_X86_64_GLOB_DAT  __libc_start_main@GLIBC_2.2.5
0000000000003fe8 R_X86_64_GLOB_DAT  __gmon_start__
0000000000003ff0 R_X86_64_GLOB_DAT  _ITM_registerTMCloneTable
0000000000003ff8 R_X86_64_GLOB_DAT  __cxa_finalize@GLIBC_2.2.5
0000000000003fb8 R_X86_64_JUMP_SLOT  __stack_chk_fail@GLIBC_2.4
0000000000003fc0 R_X86_64_JUMP_SLOT  close@GLIBC_2.2.5
0000000000003fc8 R_X86_64_JUMP_SLOT  read@GLIBC_2.2.5
0000000000003fd0 R_X86_64_JUMP_SLOT  open@GLIBC_2.2.5
```
`read@GLIBC_2.2.5`的绝对地址为0x3fc8。用gdb调试追踪一下read。

```
$ (gdb) x/8x 0x3fc8
$ (gdb) disassemble 0x00001050
$ (gdb) disassemble 0x00001060
$ (gdb) disassemble 0x00000000
```

![](/images/what_we_talk_about_when_we_talk_about_chrdev/20200918105016.png)

可以看到没有加载符号表，找不到包含指定地址的函数。进入gdb，将断点设置为main，查看共享库。
```
$ (gdb) info sharedlibrary
```

![](/images/what_we_talk_about_when_we_talk_about_chrdev/20200918105445.png)

可以看见提示说共享库缺少调试信息。根据from的地址增加so文件的符号表，增加elf的源路径。
```
$ (gdb) add-symbol-file /usr/lib/debug/lib/x86_64-linux-gnu/ld-2.31.so 0x00007ffff7fd0100
$ (gdb) add-symbol-file /usr/lib/debug/lib/x86_64-linux-gnu/ld-2.31.so 0x00007ffff7de9630
$ (gdb) dir ~/glibc/glibc-2.31/elf/
```
此时再次查看`read@plt`

![](/images/what_we_talk_about_when_we_talk_about_chrdev/20200918110222.png)

继续查看jmpq的完整地址，查看其在符号表中的对应信息。
```
$ (gdb) x/a 0x555555557fc8
$ (gdb) info symbol 0x7ffff7ed4fa0
```

![](/images/what_we_talk_about_when_we_talk_about_chrdev/20200918111050.png)

发现能在so的text section中找到，此时程序运行到read的动态链接就能完成。整个共享库的地址翻译，离不开GOT和PLT。

PLT全称Procedure Link Table(过程链接表)，GOT全称Global Offset Table(全局偏移表)，两者在代码共享及动态库上扮演着重要的角色。在二进制文件中，有一个叫relocations的东西，它能够在工具链静态链接时填充或者在运行时动态链接时填充。它在二进制文件中的作用就是确定某个符号的值，然后将这个符号的值写到某个偏移地址处。

使用readelf查看可执行文件：

![](/images/what_we_talk_about_when_we_talk_about_chrdev/20200918112553.png)

可以看到在`.rela.plt`的section下，read的符号值为0，存在一个偏移地址为`0x000000003fc8`，如果使用gdb查看这个位置的机器码的话，会得到下面的结果：

![](/images/what_we_talk_about_when_we_talk_about_chrdev/20200918113505.png)

可以看到从offset 0x2位置开始，寄存器rax的值是未确定的，此时对于%al来说，只能以一个0x00作为偏移，来读取此处内存地址的内容。然而实际上，这个寄存器的偏移地址会在链接后进行补全，最后形成真正的地址偏移。在链接阶段，编译器如果发现有代码定义在动态库，链接器会生成一小段额外代码取代原来的地址进行链接重定位，从而进行运行时的重定位。此时PLT将这个区域进行翻译，实现对另一个地址的跳转，之后通过GOT找到so中真正的地址。ELF将GOT拆分为.got和.got.plt，将PLT拆分为.plt和.plt.got，前者保存全局变量引用的地址，后者保存函数引用的地址，在ELF信息中都能看见。而对于.rela来说，.rela.dyn包含relocations动态段的填充地址，.rela.plt包含relocations的plt的偏移量，通过偏移量来找到GOT表项的地址。

PLT表结构第一项为公共表项，下面是每个动态库函数有一项，并从对应的GOT表项中读取目标函数地址。GOT表前三项为.dyn段地址、link_map数据结构地址和`_dl_runtime_resolve`函数地址，在动态链接时进行填充。进程加载后，PLT指向不变，动态链接器在ELF的.dynamic段里面找到GOT的地址，当外部函数被调用时才执行重定位。

GOT还实现了延迟重定位的功能，确保在动态库函数被调用时才作解析，还能获取重定位是否完成的标志，这就是PLT stub实现的延迟绑定技术(lazy binding)。Linux还分类了公共函数，增加.plt的公共入口，避免PLT指令过多。

## 回到read

了解了PLT和GOT之后，动态库的链接也不再那么神秘。如果把目光重新放回read，可能会发现一个函数：`new_sync_read()`。在`__vfs_read()`里面，通过对file成员f_op的文件操作函数判断，f_op有read就走read，f_op有read_iter就走`new_sync_read()`。如果我们查看`fs/ext4/file.c`、`fs/btrfs/file.c`中file_operations结构的定义，可以发现在Ext4、Btrfs这些文件系统都不再使用read，取而代之的是read_iter。

在剖析read_iter之前，先来看下这3个结构：iovec、kiocb、iov_iter。iovec定义在`include/uapi/linux/uio.h`中，是POSIX标准1003.1g定义的，需要表示io向量的基址和大小。iov_iter定义在`include/linux/uio.h`中，作为iovec结构的迭代器，处理用户空间提供的数据缓冲区，普遍用于内存管理和文件系统管理。kiocb定义在`include/linux/fs.h`中，表示kernel io control block，用来记录io操作的完成状态。在`new_sync_read()`中，先调用`init_sync_kiocb()`初始化内核io控制块，其中通过`iocb_flags()`设置状态。此时对于这个判断通常分为O_DIRECT和使用缓存，用`io_is_direct()`判断，O_DIRECT绕过内核缓存模式，直接进行磁盘的读写。
```
static inline bool io_is_direct(struct file *filp)
{
	return (filp->f_flags & O_DIRECT) || (filp->f_mapping->host->i_flags & S_DAX);
}
```
在内核io控制块初始化之后，将传入的读写地址赋给kiocb，调用`iov_iter_init()`初始化iovec迭代器。iovec中存放着用户空间传入的数据和长度，使用iov_iter减少了内核在数据缓冲区中出错的可能，通过计数、copy等原子操作修改数据。最终，`call_read_iter()`方法将kiocb、iov_iter3者传入驱动设备的read_iter操作中。

之前说过，驱动设备预留了文件操作的通用实现。对于文件系统的read_iter来说，不可避免地会用到`generic_file_read_iter()`。如果kiocb有效且不用等待，就可以在文件的偏移量起始位置进行写回直到结束。

接下来，read有两个选择，首先判断kiocb的flag是否是Direct IO方式，如果是这个方式则用`filemap_write_and_wait_range()`确保页是最新的，然后使用`mapping->a_ops->direct_IO()`来访问数据，其中direct_IO是address_space_operations结构里的函数。当遇到iovec读完、遇到EOF、返回错误码、读到DAX文件等情况，此时直接返回。

如果不是Direct IO方式的时候，调用`generic_file_buffered_read()`，进行缓存方式的文件读取。这种方式用到了页高速缓存，减少了磁盘的io操作，加快了读写速度。Linux对于内存管理采用了基数树(radix tree)的方法，每个inode有一个address_space对象，结构address_space通过radix树跟踪绑定到地址映射上的核心页。

在`generic_file_buffered_read()`中，使用了大量的goto语句来梳理逻辑和处理错误。对于一个正常的流程来说，当读取一个文件时，先要调用`find_get_page()`，检测数据是否已经缓存，如果没有缓存，则执行预读取`page_cache_sync_readahead()`、`page_cache_async_readahead()`从硬盘中读取页。如果内存中没有pagecache，则通过`page_cache_alloc_cold()`将page加入到`add_to_page_cache_lru()`，通过LRU算法将数据加载入缓存页。

在确保找到页缓存且页缓存为最新页后，调用`copy_page_to_iter()`将内存中的数据拷贝到用户空间，最后用`file_accessed()`更新文件的最后访问时间。就这样，当击中最新页缓存且iov_iter计数大于0时，拷贝缓存页的数据，当未击中页缓存时，则执行预读取或分配新的缓存页，利用goto语句完成整个迭代器对文件的读取。

![](/images/what_we_talk_about_when_we_talk_about_chrdev/chrdev1.jpg)

---
layout: post
title: "【学习Xv6】加载并运行内核"
date: 2015-05-26 14:03:28 +0800
comments: true
categories: [操作系统]
---

##前情提要

学习 xv6 系列的上一篇[《【学习xv6】从实模式到保护模式》](http://leenjewel.github.io/blog/2014/07/29/%5B%28xue-xi-xv6%29%5D-cong-shi-mo-shi-dao-bao-hu-mo-shi/)讲到我们的系统已经将计算机的 CPU 从实模式切换到保护模式状态下了，接下来我们可以暂时告别晦涩难懂的汇编语言来到 C 语言环境中了，引导的工作快要接近尾声，内核即将被载入运行。

##预备知识

###从硬盘读取数据

在[《【学习xv6】从实模式到保护模式》](http://leenjewel.github.io/blog/2014/07/29/%5B%28xue-xi-xv6%29%5D-cong-shi-mo-shi-dao-bao-hu-mo-shi/)中我们已经讲到了如何通过向 804x 键盘控制器端口发送信号来打开 A20 gate 了，同样道理，我们向硬盘控制器的指定端口发送信号就可以操作硬盘，从硬盘读取或向硬盘写入数据。IDE 标准定义了 8 个寄存器来操作硬盘。PC 体系结构将第一个硬盘控制器映射到端口 1F0-1F7 处，而第二个硬盘控制器则被映射到端口 170-177 处。这几个寄存器的描述如下（以第一个控制器为例）：

```
1F0        - 数据寄存器。读写数据都必须通过这个寄存器

1F1        - 错误寄存器，每一位代表一类错误。全零表示操作成功。

1F2        - 扇区计数。这里面存放你要操作的扇区数量

1F3        - 扇区LBA地址的0-7位

1F4        - 扇区LBA地址的8-15位

1F5        - 扇区LBA地址的16-23位

1F6 (低4位) - 扇区LBA地址的24-27位

1F6 (第4位) - 0表示选择主盘，1表示选择从盘

1F6 (5-7位) - 必须为1

1F7 (写)    - 命令寄存器

1F7 (读)    - 状态寄存器

              bit 7 = 1  控制器忙
              bit 6 = 1  驱动器就绪
              bit 5 = 1  设备错误
              bit 4        N/A
              bit 3 = 1  扇区缓冲区错误
              bit 2 = 1  磁盘已被读校验
              bit 1        N/A
              bit 0 = 1  上一次命令执行失败
```

稍后讲到从硬盘加载内核到内存时我们再通过 xv6 的实际代码来看看硬盘操作的具体实现。

###ELF文件格式

在[ Wiki 百科上有 ELF 文件格式的详细解释](http://en.wikipedia.org/wiki/Executable_and_Linkable_Format)，简单的说 ELF 文件格式是 Linux 下可执行文件的标准格式。就好像 Windows 操作系统里的可执行文件 .exe 一样（当然，Windows 里的可执行文件的标准格式叫 PE 文件格式），Linux 操作系统里的可执行文件也有它自己的格式。只有按照文件标准格式组织好的可执行文件操作系统才知道如何加载运行它。我们并使使用 C 语言按照教科书写出的 HelloWorld 代码在 Linux 环境下最终通过编译器（gcc等）编译出的可以运行的程序就是 ELF 文件格式的。

那么 ELF 文件格式具体的结构是怎样的呢？ 大概是下面这个样子的。

| ELF 头部 ( ELF Header ) |
|:-----------------------:|
| 程序头表 (Program Header Table) 
|.text
|.rodata
|......
|节头表 (Section Header Table)

这里我们暂时只关心 ELF 文件结构的前两个部分：ELF 头部和程序头表，xv6 源代码的 elf.h 文件中有其详细的定义，我们来看一下。

```c
#define ELF_MAGIC 0x464C457FU  // "\x7FELF" in little endian

// ELF 文件格式的头部
struct elfhdr {
  uint magic;       // 4 字节，为 0x464C457FU（大端模式）或 0x7felf（小端模式）
  				    // 表明该文件是个 ELF 格式文件

  uchar elf[12];    // 12 字节，每字节对应意义如下：
                    //     0 : 1 = 32 位程序；2 = 64 位程序
                    //     1 : 数据编码方式，0 = 无效；1 = 小端模式；2 = 大端模式
                    //     2 : 只是版本，固定为 0x1
                    //     3 : 目标操作系统架构
                    //     4 : 目标操作系统版本
                    //     5 ~ 11 : 固定为 0
                    
  ushort type;      // 2 字节，表明该文件类型，意义如下：
                    //     0x0 : 未知目标文件格式
                    //     0x1 : 可重定位文件
                    //     0x2 : 可执行文件
                    //     0x3 : 共享目标文件
                    //     0x4 : 转储文件
                    //     0xff00 : 特定处理器文件
                    //     0xffff : 特定处理器文件

  ushort machine;   // 2 字节，表明运行该程序需要的计算机体系架构，
                    // 这里我们只需要知道 0x0 为未指定；0x3 为 x86 架构

  uint version;     // 4 字节，表示该文件的版本号

  uint entry;       // 4 字节，该文件的入口地址，没有入口（非可执行文件）则为 0

  uint phoff;       // 4 字节，表示该文件的“程序头部表”相对于文件的位置，单位是字节

  uint shoff;       // 4 字节，表示该文件的“节区头部表”相对于文件的位置，单位是字节

  uint flags;       // 4 字节，特定处理器标志

  ushort ehsize;    // 2 字节，ELF文件头部的大小，单位是字节

  ushort phentsize; // 2 字节，表示程序头部表中一个入口的大小，单位是字节

  ushort phnum;     // 2 字节，表示程序头部表的入口个数，
                    // phnum * phentsize = 程序头部表大小（单位是字节）

  ushort shentsize; // 2 字节，节区头部表入口大小，单位是字节

  ushort shnum;     // 2 字节，节区头部表入口个数，
                    // shnum * shentsize = 节区头部表大小（单位是字节）

  ushort shstrndx;  // 2 字节，表示字符表相关入口的节区头部表索引
};

// 程序头表
struct proghdr {
  uint type;        // 4 字节， 段类型
                    //         1 PT_LOAD : 可载入的段
                    //         2 PT_DYNAMIC : 动态链接信息
                    //         3 PT_INTERP : 指定要作为解释程序调用的以空字符结尾的路径名的位置和大小
                    //         4 PT_NOTE : 指定辅助信息的位置和大小
                    //         5 PT_SHLIB : 保留类型，但具有未指定的语义
                    //         6 PT_PHDR : 指定程序头表在文件及程序内存映像中的位置和大小
                    //         7 PT_TLS : 指定线程局部存储模板
  uint off;         // 4 字节， 段的第一个字节在文件中的偏移
  uint vaddr;       // 4 字节， 段的第一个字节在内存中的虚拟地址
  uint paddr;       // 4 字节， 段的第一个字节在内存中的物理地址(适用于物理内存定位型的系统)
  uint filesz;      // 4 字节， 段在文件中的长度
  uint memsz;       // 4 字节， 段在内存中的长度
  uint flags;       // 4 字节， 段标志
                    //         1 : 可执行
                    //         2 : 可写入
                    //         4 : 可读取
  uint align;       // 4 字节， 段在文件及内存中如何对齐
};
```

###ELF文件的加载与运行

既然 ELF 标准文件格式是可执行文件（当然不仅仅用于可执行文件，还可以用于动态链接库文件等）使用的文件格式，那么它一定是可以被加载并运行的。学习 xv6 系列的上一篇[《【学习xv6】从实模式到保护模式》](http://leenjewel.github.io/blog/2014/07/29/%5B%28xue-xi-xv6%29%5D-cong-shi-mo-shi-dao-bao-hu-mo-shi/)的预备知识中我们讲到

>程序的组成我们可以简单的理解为：数据加上指令就是程序。

我们写好的程序代码经过编译器的编译成为机器码，而机器码根据其自身的作用不同被分为不同的段，其中最主要的就是**代码段**和**数据段**。

而一个可执行程序又是有很多个这样的段组成的，一个可执行程序可以有好几个代码段和好几个数据段和其他不同的段。当一个程序准备运行的时候，操作系统会将程序的这些段载入到内从中，再通知 CPU 程序代码段的位置已经开始执行指令的点即入口点。

既然一个可执行程序有多个代码段、多个数据段和其他段，操作系统在加载这些段的时候为了更好的组织利用内存，希望将一些列作用相同的段放在一起加载（比如多个代码段就可以一并加载），编译器为了方便操作系统加载这些作用相同的段，在编译的时候会刻意将作用相同的段安排在一起。而这些作用相同的段在程序中（ELF文件）中是如何组织的，这些组织信息就被记录在 ELF 文件的程序头表中。

所以一个 ELF 文件格式的可执行程序的加载运行过程是这样的：

* 通过读取 ELF 头表中的信息了解该可执行程序是否可以运行（版本号，适用的计算机架构等等）
* 通过 ELF 头表中的信息找到程序头表
* 通过读取 ELF 文件中程序头表的信息了解可执行文件中各个段的位置以及加载方式
* 将可执行文件中需要加载的段加载到内存中，并通知 CPU 从指定的入口点开始执行

##从 bootmain 开始

学习 xv6 系列的上一篇[《【学习xv6】从实模式到保护模式》](http://leenjewel.github.io/blog/2014/07/29/%5B%28xue-xi-xv6%29%5D-cong-shi-mo-shi-dao-bao-hu-mo-shi/)的最后我们写到

> 通过这个跳转实际上 CPU 就会跳转到 bootasm.S 文件的 start32 标识符处继续执行了

我们打开 bootasm.S 文件看看对应的 start32 位置处的代码做了什么事情。

```
start32:
  # Set up the protected-mode data segment registers
  # 像上面讲 ljmp 时所说的，这时候已经在保护模式下了
  # 数据段在 GDT 中的下标是 2，所以这里数据段的段选择子是 2 << 3 = 0000 0000 0001 0000
  # 这 16 位的段选择子中的前 13 位是 GDT 段表下标，这里前 13 位的值是 2 代表选择了数据段
  # 这里将 3 个数据段寄存器都赋值成数据段段选择子的值
  movw    $(SEG_KDATA<<3), %ax    # Our data segment selector  段选择子赋值给 ax 寄存器
  movw    %ax, %ds                # -> DS: Data Segment        初始化数据段寄存器
  movw    %ax, %es                # -> ES: Extra Segment       初始化扩展段寄存器
  movw    %ax, %ss                # -> SS: Stack Segment       初始化堆栈段寄存器
  movw    $0, %ax                 # Zero segments not ready for use  ax 寄存器清零
  movw    %ax, %fs                # -> FS                      辅助寄存器清零
  movw    %ax, %gs                # -> GS                      辅助寄存器清零

  # Set up the stack pointer and call into C.
  movl    $start, %esp
  call    bootmain
```

这里在初始化了一些寄存器后直接调用了一个叫做 `bootmain` 的函数，而这个函数是写在 bootmain.c 文件中的，终于我们暂时告别了汇编来到了 C 的世界了。来看看 bootmain 函数在做什么事情。

##载入内核

bootmain.c 这个文件很小，代码很少，它其实是引导工作的最后部分（引导的大部分工作都在 bootasm.S 中实现），它负责将内核从硬盘上加载到内存中，然后开始执行内核中的程序。我们来看代码。

```c
#define SECTSIZE  512  // 硬盘扇区大小 512 字节

void
bootmain(void)
{
  struct elfhdr *elf;
  struct proghdr *ph, *eph;
  void (*entry)(void);
  uchar* pa;

  // 从 0xa0000 到 0xfffff 的物理地址范围属于设备空间，
  // 所以内核放置在 0x10000 处开始
  elf = (struct elfhdr*)0x10000;  // scratch space

  // 从内核所在硬盘位置读取一内存页 4kb 数据
  readseg((uchar*)elf, 4096, 0);

  // 判断是否为 ELF 文件格式
  if(elf->magic != ELF_MAGIC)
    return;  // let bootasm.S handle error

  // 加载 ELF 文件中的程序段 (ignores ph flags).
  ph = (struct proghdr*)((uchar*)elf + elf->phoff);
  eph = ph + elf->phnum;
  for(; ph < eph; ph++){
    pa = (uchar*)ph->paddr;
    readseg(pa, ph->filesz, ph->off);
    if(ph->memsz > ph->filesz)
      stosb(pa + ph->filesz, 0, ph->memsz - ph->filesz);
  }

  // Call the entry point from the ELF header.
  // Does not return!
  entry = (void(*)(void))(elf->entry);
  entry();
}
```

这里将内核（一个 ELF 格式文件）从硬盘读取到内存 `0x10000` 处的关键方法是 `readseg(uchar*, uint, uint)` 我们再来看看这个函数的具体实现代码

```c
void
readseg(uchar* pa, uint count, uint offset)  // 0x10000, 4096(0x1000), 0
{
  uchar* epa;

  epa = pa + count;  // 0x11000

  // 根据扇区大小 512 字节做对齐
  pa -= offset % SECTSIZE;

  // bootblock 引导区在第一扇区（下标为 0），内核在第二个扇区（下标为 1）
  // 这里做 +1 操作是统一略过引导区
  offset = (offset / SECTSIZE) + 1;

  // If this is too slow, we could read lots of sectors at a time.
  // We'd write more to memory than asked, but it doesn't matter --
  // we load in increasing order.
  // 一次读取一个扇区 512 字节的数据
  for(; pa < epa; pa += SECTSIZE, offset++)
    readsect(pa, offset);
}
```

我们来看看为什么说内核在磁盘的第二扇区，引导区在磁盘的第一扇区。在 xv6 系列文章的第一篇[《【学习 Xv6 】在 Mac OSX 下运行 Xv6》](http://leenjewel.github.io/blog/2014/07/24/%5B%28xue-xi-xv6-%29%5D-zai-mac-osx-xia-yun-xing-xv6/)里讲到过

> 编译成功后我们会得到 xv6.img 和 fs.img 两个文件。
> 
> 在 Hardware 配置页的 Hard disk 里把 xv6.img 载入进去。
> 
> 在 Advanced 配置页的 Hard disk 2 里把 fs.img 载入进去。

由此我们可以猜测内核应该在 xv6.img 这个镜像文件中。下面我们通过 Makefile 来印证这一点，我们看一下 xv6 的 Makefile 文件关于 xv6.img 构建过程的说明

```
xv6.img: bootblock kernel fs.img
	dd if=/dev/zero of=xv6.img count=10000
	dd if=bootblock of=xv6.img conv=notrunc
	dd if=kernel of=xv6.img seek=1 conv=notrunc
```

可以看出 xv6.img 是一个由 10000 个扇区组成的（512b x 10000 = 5 MB），而里面包含的只有 `bootblock` 和 `kernel` 两个块，通过名字我们不难看出 `bootblock` 就是引导区，它的大小正好是 512 字节即一个磁盘扇区大小（可以通过文件浏览器看到），所以根据它们写入 xv6.img 的顺序我们证实了猜测，在 xv6 系统中引导区占一个磁盘扇区大小，放置在磁盘的第一扇区，紧随其后的是内核文件（ELF 文件格式）。我们用一个十六进制编辑器打开 kernel 文件看看，可以看到开头的数据内如如下

|magic|elf[12]|type|machine|version|entry|phoff|shoff|flags|
|:---:|:-----:|:--:|:-----:|:-----:|:---:|:---:|:---:|:---:|
|7F 45 4C 46|01 01 01 00 00 00 00 00 00 00 00 00|02 00|03 00|01 00 00 00|0C 00 10 00|34 00 00 00|00 F6 01 00|00 00 00 00|
|ehsize|phentsize|phnum|shentsize|shnum|shstrndx|
|34 00|20 00|02 00|28 00|12 00|0F 00|

而内核文件的前 4 字节正式 ELF 文件头的模数 `ELF_MAGIC 0x464C457F` 这也说明了内核文件确实是一个 ELF 格式的文件。如果我们按照 ELF 文件结构重拍上面的机器码会是这样

|字段名称|大小|数值|意义|
|:-----:|:-:|:-:|:-:|
|magic|4字节|7F 45 4C 46|ELF 格式文件|
|elf|12字节|01 01 01 00 00 00 00 00 00 00 00 00| 32 位小端模式，目标操作系统为 System V
|type|2字节|02 00|可执行文件|
|machine|2字节|03 00|指定计算机体系架构为 x86|
|version|4字节|01 00 00 00|版本号为 1|
|entry|4字节|0C 00 10 00|该可执行文件入口地址|
|phoff|4字节|34 00 00 00|程序头表相对于文件的起始位置是 52 字节|
|shoff|4字节|00 F6 01 00|节区头表相对于文件的起始位置是 128512 字节|
|flags|4字节|00 00 00 00|无特定处理器标志|
|ehsize|2字节|34 00|ELF 头大小为 52 字节|
|phentsize|2字节|20 00|程序头表一个入口的大小是 32 字节|
|phnum|2字节|02 00|程序头表入口个数是 2 个|
|shentsize|2字节|28 00|节区头表入口大小是 40 字节|
|shnum|2字节|12 00|节区头表入口个数是 18 个|
|shstrndx|2字节|0F 00|字符表入口在节区头表的索引是 15|

通过十六进制编辑器逐个字节的去分析内核文件的 ELF 头部是希望大家能有个更直观的认识，当然了 Linux 也为我们提供了方便的工具 `readelf` 命令来检查 ELF 文件的相关信息。我们再通过 `readelf` 命令验证一下我们刚刚通过十六进制编辑器分析的结果。

```sh
$ readelf -h kernel
ELF Header:
  Magic:   7f 45 4c 46 01 01 01 00 00 00 00 00 00 00 00 00
  Class:                             ELF32
  Data:                              2's complement, little endian
  Version:                           1 (current)
  OS/ABI:                            UNIX - System V
  ABI Version:                       0
  Type:                              EXEC (Executable file)
  Machine:                           Intel 80386
  Version:                           0x1
  Entry point address:               0x10000c
  Start of program headers:          52 (bytes into file)
  Start of section headers:          128512 (bytes into file)
  Flags:                             0x0
  Size of this header:               52 (bytes)
  Size of program headers:           32 (bytes)
  Number of program headers:         2
  Size of section headers:           40 (bytes)
  Number of section headers:         18
  Section header string table index: 15
```

最后我们看一下从磁盘读取内核到内存的方法实现，看看是怎样通过向特定端口发送数据来达到操作磁盘目的的。具体的说明请看代码附带的注释。

```c
// Read a single sector at offset into dst.
// 这里使用的是 LBA 磁盘寻址模式
// LBA是非常单纯的一种寻址模式﹔从0开始编号来定位区块，
// 第一区块LBA=0，第二区块LBA=1，依此类推
void
readsect(void *dst, uint offset)      // 0x10000, 1
{
  // Issue command.
  waitdisk();
  outb(0x1F2, 1);                     // 要读取的扇区数量 count = 1
  outb(0x1F3, offset);                // 扇区 LBA 地址的 0-7 位
  outb(0x1F4, offset >> 8);           // 扇区 LBA 地址的 8-15 位
  outb(0x1F5, offset >> 16);          // 扇区 LBA 地址的 16-23 位
  outb(0x1F6, (offset >> 24) | 0xE0); // offset | 11100000 保证高三位恒为 1
                                      //         第7位     恒为1
                                      //         第6位     LBA模式的开关，置1为LBA模式
                                      //         第5位     恒为1
                                      //         第4位     为0代表主硬盘、为1代表从硬盘
                                      //         第3~0位   扇区 LBA 地址的 24-27 位
  outb(0x1F7, 0x20);                  // 20h为读，30h为写

  // Read data.
  waitdisk();
  insl(0x1F0, dst, SECTSIZE/4);
}
```

##运行内核

内核从磁盘上载入到内存中后 `bootmain` 函数接下来就准备运行内核中的方法了。我们还是回到 `bootmain` 函数上来，请注意看我加上的注释说明。

```c
void
bootmain(void)
{
  struct elfhdr *elf;
  struct proghdr *ph, *eph;
  void (*entry)(void);
  uchar* pa;

  // 从 0xa0000 到 0xfffff 的物理地址范围属于设备空间，
  // 所以内核放置在 0x10000 处开始
  elf = (struct elfhdr*)0x10000;

  // 从内核所在硬盘位置读取一内存页 4kb 数据
  readseg((uchar*)elf, 4096, 0);

  // 判断是否为 ELF 文件格式
  if(elf->magic != ELF_MAGIC)
    return;  // let bootasm.S handle error

  // 加载 ELF 文件中的程序段 (ignores ph flags).
  // 找到内核 ELF 文件的程序头表
  ph = (struct proghdr*)((uchar*)elf + elf->phoff);
  // 内核 ELF 文件程序头表的结束位置
  eph = ph + elf->phnum;
  // 开始将内核 ELF 文件程序头表载入内存
  for(; ph < eph; ph++){
    pa = (uchar*)ph->paddr;
    readseg(pa, ph->filesz, ph->off);
    // 如果内存大小大于文件大小，用 0 补齐内存空位
    if(ph->memsz > ph->filesz)
      stosb(pa + ph->filesz, 0, ph->memsz - ph->filesz);
  }

  // Call the entry point from the ELF header.
  // Does not return!
  // 从内核 ELF 文件入口点开始执行内核
  entry = (void(*)(void))(elf->entry);
  entry();
}
```

载入内核后根据 ELF 头表的说明，`bootmain`函数开始将内核 ELF 文件的程序头表从磁盘载入内存，为运行内核代码做着最后的准备工作。根据上一节的分析我们知道内核的 ELF 文件的程序头表紧跟在 ELF 头表后面，程序头表一共 2 个，每个 32 字节大小，一共是 64 字节，我们继续用十六进制编辑器打开 `kernel` 内核二进制文件看看程序头表的内容。

|type|off|vaddr|paddr|filesz|memsz|flags|align|
|:---------:|:---------:|:---------:|:---------:|:---------:|:---------:|:---------:|:---------:|
|01 00 00 00|00 10 00 00|00 00 10 80|00 00 10 00|96 B5 00 00|FC 26 01 00|07 00 00 00|00 10 00 00|
|type|off|vaddr|paddr|filesz|memsz|flags|align|
|51 E5 74 64|00 00 00 00|00 00 00 00|00 00 00 00|00 00 00 00|00 00 00 00|07 00 00 00|04 00 00 00|

* 程序头表 1

|字段名称|大小|数值|意义|
|:-----:|:-:|:-:|:-:|
|type|4字节|01 00 00 00|可载入的段|
|off|4字节|00 10 00 00|段在文件中的偏移是 4096 字节|
|vaddr|4字节|00 00 10 80|段在内存中的虚拟地址|
|paddr|4字节|00 10 00 00|同段在文件中的偏移量|
|filesz|4字节|96 B5 00 00|段在文件中的大小是 46486 字节|
|memsz|4字节|FC 26 01 00|段在内存中的大小是 75516 字节|
|flags|4字节|07 00 00 00|段的权限是可写、可读、可执行|
|align|4字节|00 10 00 00|段的对齐方式是 4096 字节，即4kb|

* 程序头表 2

|字段名称|大小|数值|意义|
|:-----:|:-:|:-:|:-:|
|type|4字节|51 E5 74 64| PT_GNU_STACK 段|
|off|4字节|00 00 00 00|段在文件中的偏移是 0 字节|
|vaddr|4字节|00 00 00 00|段在内存中的虚拟地址|
|paddr|4字节|00 00 00 00|同段在文件中的偏移量|
|filesz|4字节|00 00 00 00|段在文件中的大小是 0 字节|
|memsz|4字节|00 00 00 00|段在内存中的大小是 0 字节|
|flags|4字节|07 00 00 00|段的权限是可写、可读、可执行|
|align|4字节|04 00 00 00|段的对齐方式是 4 字节|

同样我们再通过 `readelf` 命令来验证我们通过十六进制编辑器对内核 ELF 文件的程序头表的分析结果十分正确。

```sh
readelf -l kernel

Elf file type is EXEC (Executable file)
Entry point 0x10000c
There are 2 program headers, starting at offset 52

Program Headers:
  Type           Offset   VirtAddr   PhysAddr   FileSiz MemSiz  Flg Align
  LOAD           0x001000 0x80100000 0x00100000 0x0b596 0x126fc RWE 0x1000
  GNU_STACK      0x000000 0x00000000 0x00000000 0x00000 0x00000 RWE 0x4

 Section to Segment mapping:
  Segment Sections...
   00     .text .rodata .stab .stabstr .data .bss
   01
```

在预备知识里我们讲到 ELF 文件的程序头表描述了程序各个段的情况，所以我们再通过`readelf`命令看看内核文件都有那些段

```sh
readelf -S kernel
There are 18 section headers, starting at offset 0x1f600:

Section Headers:
  [Nr] Name              Type            Addr     Off    Size   ES Flg Lk Inf Al
  [ 0]                   NULL            00000000 000000 000000 00      0   0  0
  [ 1] .text             PROGBITS        80100000 001000 008111 00  AX  0   0  4
  [ 2] .rodata           PROGBITS        80108114 009114 000672 00   A  0   0  4
  [ 3] .stab             PROGBITS        80108786 009786 000001 0c  WA  4   0  1
  [ 4] .stabstr          STRTAB          80108787 009787 000001 00  WA  0   0  1
  [ 5] .data             PROGBITS        80109000 00a000 002596 00  WA  0   0 4096
  [ 6] .bss              NOBITS          8010b5a0 00c596 00715c 00  WA  0   0 32
  [ 7] .debug_line       PROGBITS        00000000 00c596 001f8c 00      0   0  1
  [ 8] .debug_info       PROGBITS        00000000 00e522 00a965 00      0   0  1
  [ 9] .debug_abbrev     PROGBITS        00000000 018e87 0026ed 00      0   0  1
  [10] .debug_aranges    PROGBITS        00000000 01b578 0003a0 00      0   0  8
  [11] .debug_loc        PROGBITS        00000000 01b918 002f30 00      0   0  1
  [12] .debug_str        PROGBITS        00000000 01e848 000cdc 01  MS  0   0  1
  [13] .comment          PROGBITS        00000000 01f524 00001c 01  MS  0   0  1
  [14] .debug_ranges     PROGBITS        00000000 01f540 000018 00      0   0  1
  [15] .shstrtab         STRTAB          00000000 01f558 0000a5 00      0   0  1
  [16] .symtab           SYMTAB          00000000 01f8d0 0023d0 10     17 138  4
  [17] .strtab           STRTAB          00000000 021ca0 0012d0 00      0   0  1
Key to Flags:
  W (write), A (alloc), X (execute), M (merge), S (strings)
  I (info), L (link order), G (group), T (TLS), E (exclude), x (unknown)
  O (extra OS processing required) o (OS specific), p (processor specific)
```

结合这两次 `readelf` 命令的输出我们不难看出，内核文件的 ELF 程序头表中只有第一个是需要被加载的，而这个程序头表指出的加载位置 `0x80100000` 和内核程序的代码段 `.text` 的位置是一样的。

而要加载的段是 `.text .rodata .stab .stabstr .data .bss` ，这些段在内存中的大小总和是 `0x008111 + 0x000672 + 0x000001 + 0x000001 + 0x002596 + 0x00715c = 0x73335` 按照对齐要求 `0x1000` 对齐后为 `0x75516` 和 ELF 程序头表中的内存大小信息一致。

我们再算算这些段在文件中的大小，由于这些段在文件中是顺序排列的，所以用 `.bss段` 的文件偏移量减去 `.text段` 的文件偏移量 `0x00c596 - 0x001000 = 46486` 这也是和 ELF 程序头表中段在文件中大小的信息一致。

##内核加载后的系统内存布局

至此内核已经被载入内存并准备投入运行了。在结束这一篇前我们再看一眼目前状态下系统整体的内存布局，对即将运行的内核环境有一个大致的了解。我们来看几个关键点

```c
// bootmain.c

void
bootmain(void)
{
  struct elfhdr *elf;
  struct proghdr *ph, *eph;
  void (*entry)(void);
  uchar* pa;

  // 从 0xa0000 到 0xfffff 的物理地址范围属于设备空间，
  // 所以内核放置在 0x10000 处开始
  elf = (struct elfhdr*)0x10000;

  // 从内核所在硬盘位置读取一内存页 4kb 数据
  readseg((uchar*)elf, 4096, 0);

  // 省略后面的代码......
}
```

由此可知内核被放置在 0x10000 处开始。

```nasm
# bootasm.S

.code32  # Tell assembler to generate 32-bit code now.
start32:
  # Set up the protected-mode data segment registers
  # 像上面讲 ljmp 时所说的，这时候已经在保护模式下了
  # 数据段在 GDT 中的下标是 2，所以这里数据段的段选择子是 2 << 3 = 0000 0000 0001 0000
  # 这 16 位的段选择子中的前 13 位是 GDT 段表下标，这里前 13 位的值是 2 代表选择了数据段
  # 这里将 3 个数据段寄存器都赋值成数据段段选择子的值
  movw    $(SEG_KDATA<<3), %ax    # Our data segment selector  段选择子赋值给 ax 寄存器
  movw    %ax, %ds                # -> DS: Data Segment        初始化数据段寄存器
  movw    %ax, %es                # -> ES: Extra Segment       初始化扩展段寄存器
  movw    %ax, %ss                # -> SS: Stack Segment       初始化堆栈段寄存器
  movw    $0, %ax                 # Zero segments not ready for use  ax 寄存器清零
  movw    %ax, %fs                # -> FS                      辅助寄存器清零
  movw    %ax, %gs                # -> GS                      辅助寄存器清零

  # Set up the stack pointer and call into C.
  movl    $start, %esp            # 栈顶被放置在 0x7C00 处，即 $start
  call    bootmain

```

由此可知在执行 `bootmain.c` 之前 `bootasm.S` 汇编代码已经将栈的栈顶设置在了 `0x7C00` 处。之前我们了解过 x86 架构计算机的启动过程，BIOS 会将引导扇区的引导程序加载到 `0x7C00` 处并引导 CPU 从此处开始运行，故栈顶即被设置在了和引导程序一致的内存位置上。我们知道栈是自栈顶开始向下增长的，所以这里栈会逐渐远离引导程序，所以这里这样安置栈顶的位置并无什么问题。

最后放一张简单的内存布局示意图

```
0x00000000
+------------------------------------------------------------------------—+
|        0x7c00      0x7d00         0x10000                               |
|    栈    |  引导程序  |                |    内核                          |
+-------------------------------------------------------------------------+
                                                                 0xffffffff
```
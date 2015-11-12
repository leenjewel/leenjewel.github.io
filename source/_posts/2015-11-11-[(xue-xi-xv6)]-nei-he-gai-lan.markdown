---
layout: post
title: "【学习Xv6】 内核概览"
date: 2015-11-11 16:28:13 +0800
comments: true
categories: [操作系统]
---

##前情提要

上一篇[《【学习Xv6】加载并运行内核》](http://leenjewel.github.io/blog/2015/05/26/%5B%28xue-xi-xv6%29%5D-jia-zai-bing-yun-xing-nei-he/)讲到内核已经成功加载到内存中开始运行了。内核可以说是一个操作系统最核心的部件了，所以涉及要讲的内容非常非常多，我们先缓一缓脚步，对内核有一个大致的了解然后在细细的去品味它。

##内核的组成

要想知道内核里都有些什么还是要从 `Makefile` 入手看看组成内核都使用了那些源码文件。

```
kernel: $(OBJS) entry.o entryother initcode kernel.ld
    $(LD) $(LDFLAGS) -T kernel.ld -o kernel entry.o $(OBJS) -b binary initcode entryother
    $(OBJDUMP) -S kernel > kernel.asm
    $(OBJDUMP) -t kernel | sed '1,/SYMBOL TABLE/d; s/ .* / /; /^$$/d' > kernel.sym
```

而 `$(OBJS)` 变量由于内容较多我们只列出其中的一部分

```
OBJS = \
	bio.o\
	console.o\
	exec.o\
	file.o\
	fs.o\
	ide.o\
	ioapic.o\
	kalloc.o\
	......
```

从这些 `.o` 文件的文件名不难看出这些都是内核基本功能的组成部分，这也是我们以后研究的重点，既然这篇是概览我们暂时先不去关心这些组件。

除去组件剩下的文件就只有这几个：`entry.S`、`initcode.S` 和 `entryother.S` 而这几个文件我们要从哪一个先入手呢？这要听 `kernel.ld` 文件的，因为链接器在链接生成最终的内核时也是听 `kernel.ld` 文件的安排的。

`.ld` 文件是链接器配置文件或者叫链接脚本，它有自己的一套语法，链接器最终会根据链接器配置文件中的规则来生成最终的二进制文件。这里我们就不做具体的语法介绍了，有兴趣的同学请自行 Google 吧，我们只解释一下几个关键点

```
/* Simple linker script for the JOS kernel.
   See the GNU ld 'info' manual ("info ld") to learn the syntax. */

OUTPUT_FORMAT("elf32-i386", "elf32-i386", "elf32-i386")
OUTPUT_ARCH(i386)
ENTRY(_start)

SECTIONS
{
	/* Link the kernel at this address: "." means the current address */
        /* Must be equal to KERNLINK */
	. = 0x80100000;

	.text : AT(0x100000) {
		*(.text .stub .text.* .gnu.linkonce.t.*)
	}

    	/* Adjust the address for the data segment to the next page */
	. = ALIGN(0x1000);

    // ......
}
```

这里只关注四点：

- `ENTRY(_start)` 　内核的代码为段执行入口：_start
- `. = 0x80100000` 内核的起始虚拟地址位置为：0x80100000
- `.text : AT(0x100000)` 内核代码段的内存装载地址为：0x100000
- `. = ALIGN(0x1000)` 内核代码段保证 4KB 对齐

关于内核起始虚拟地址的问题我们后面遇到了再来说，代码段内存装载地址 `0x100000` 是不是看着眼熟？没错了，我们在上一篇[《【学习Xv6】加载并运行内核》](http://leenjewel.github.io/blog/2015/05/26/%5B%28xue-xi-xv6%29%5D-jia-zai-bing-yun-xing-nei-he/)最后讲到 `bootmain.c` 文件加载并运行内核时看到过，这里再把上一篇的代码列出来大家回顾一下：

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

正式因为链接脚本强制规定了内核代码段在内存中的位置才保证了引导区程序可以顺利的按照约定的地址去引导 CPU 执行内核代码。

##内核引导

既然知道了内核的入口点是 `_start` 通过简单的文本搜索不难找到这个入口点在 `entry.S` 文件中，我们直接先列出代码：

```
#include "asm.h"
#include "memlayout.h"
#include "mmu.h"
#include "param.h"

# Multiboot header.  Data to direct multiboot loader.
.p2align 2
.text
.globl multiboot_header
multiboot_header:
  #define magic 0x1badb002
  #define flags 0
  .long magic
  .long flags
  .long (-magic-flags)

# By convention, the _start symbol specifies the ELF entry point.
# Since we haven't set up virtual memory yet, our entry point is
# the physical address of 'entry'.
.globl _start
_start = V2P_WO(entry)

# Entering xv6 on boot processor, with paging off.
.globl entry
entry:
  # Turn on page size extension for 4Mbyte pages
  movl    %cr4, %eax
  orl     $(CR4_PSE), %eax
  movl    %eax, %cr4
  # Set page directory
  movl    $(V2P_WO(entrypgdir)), %eax
  movl    %eax, %cr3
  # Turn on paging.
  movl    %cr0, %eax
  orl     $(CR0_PG|CR0_WP), %eax
  movl    %eax, %cr0

  # Set up the stack pointer.
  movl $(stack + KSTACKSIZE), %esp

  # Jump to main(), and switch to executing at
  # high addresses. The indirect call is needed because
  # the assembler produces a PC-relative instruction
  # for a direct jump.
  mov $main, %eax
  jmp *%eax

.comm stack, KSTACKSIZE
```

这里我们直接略去 `multiboot_header` 这部分。这部分是为了方便通过 [GNU GRUB](https://zh.wikipedia.org/zh/GNU_GRUB) 来引导 xv6 系统的。我们直接看 `.globl _start` 部分，入口只做了一件事儿就是转到 `entry` 段继续执行。不过别看这里只有一行代码，我们也要说一下。

`V2P_WO` 的定义在 `memlayout.h` 文件中：

```
#define KERNBASE 0x80000000         // First kernel virtual address
#define V2P_WO(x) ((x) - KERNBASE)
```

它的作用是将内存虚拟地址转换成物理地址。我们现在知道内核的虚拟地址起始于 `0x80100000` 而对应的内存物理地址是 `0x100000` ，因为代码的偏移量是一样的即：

```
指令虚拟地址 = 0x80100000 + 偏移量
指令内存地址 = 0x100000 + 偏移量

所以运用初中解方程式的知识

执行内存地址 = 0x100000 + 指令虚拟地址 - 0x80100000
             = 指令虚拟地址 - 0x80000000
```

接下来我们继续看 `entry` 段做了什么。总结来说一共做了五件事儿：

- 1、开启 4MB 内存页支持
- 2、建立内存页表
- 3、开启内存分页机制
- 4、设置内核栈顶位置
- 5、跳转到 `main` 继续执行

我们一件一件的说。

###开启 4MB 内存分页支持

这是通过设置寄存器 `cr4` 的 `PSE` 位来完成的。`cr4` 寄存器是个 32 位的寄存器目前只用到低 21 位，每一位的至位都控制这一些功能的状态，所以 `cr4` 寄存器又叫做控制寄存器。

`PSE` 位是 `cr4` 控制寄存器的第 5 位，当该位置为 1 时表示内存页大小为 4MB，当置为 0 时表示内存页大小为 4KB。

###建立内存页表

这里先从内存的分页机制说起。之前我们已经接触过内存的分段管理机制了，和分段机制一样，分页机制同样是管理内存的一种方式，只不过这种方式相对于分段式来说更为先进，也是目前主流的现代操作系统所使用的内存管理方式。

通过分页将虚拟地址转换为物理地址这项工作是由 MMU(内存管理单元)负责的，以 x86 32 位架构来说它支持两级分页（Pentium Pro下分页可以是三级），这也是由 MMU 决定的。同时 x86 架构支持 4KB、2MB 和 4MB 单位页面大小的分页。当然无论以多少级进行分页，分页机制的原理是相通的，我们就以两级分页来说。

看下图是二级页表分页机制下虚拟地址转换为物理地址的过程，以 32 位系统为例我们知道 32 位系统的内存虚拟地址是 32 位的，这里我们先假设页面大小是 4KB （随后我们再说 4MB 页面的情况）。

在 4KB 页面大小情况下 32 位的虚拟地址被分为三个部分，从高位到低位分别是：10 位的一级页表索引，10 位的二级页表索引，12 位的页内偏移量。

`cr3` 寄存器中保存着一级页表所在的内存物理地址，当给出一个虚拟地址后，根据 `cr3` 的地址首先找到一级页表在内存上的存放位置。上面我们说到虚拟地址的高 10 位为一级页表的索引，所以 `2^10 = 1024` 即一级页表一共有 1024 个元素，通过虚拟地址高 10 位的索引我们找到其中一个元素，而这个元素的值指向二级页表在内存中的物理地址。

同理，虚拟地址中紧挨着高 10 位后面的 10 位是二级页表索引，因此二级页表也有 1024 个元素，通过虚拟地址的二级页表索引找到其中的一个元素，该元素指向内存分页中的一个页的地址。

通过二级页表我们现在找到了内存上的一页物理页。根据现在的设定，一物理页的大小是 4KB，4KB 的内存上还是存在着很多不同的数据的，那么我们如何从这段 4KB 内存上取得我们想要的数据呢？别忘了在虚拟地址上还剩下低 12 位没用呢，`2^12 = 4096 = 4KB` 根据这最后 12 位的索引我们最终在内存上准确的找到了我们想要的数据。

而根据二级页表和每页内存的大小我们也不难算出：

```
1024个一级页表项 x 1024个二级页表项 x 4KB页面大小 = 4GB
```

正好是 32 位系统的最大内存寻址。

我们再通过下面这张示意图体会一下这个过程。

```
寄存器                       虚拟地址
+---+        +---------------------------------------+
|cr3|        |  Page Dict  |  Page Table  |  Offset  |
+---+        |  31 -- 22   |  21 -- 12    |  11 -- 0 |
  |          +---------------------------------------+
__/                   |                   |       |-----------|
|  +--------+         |      +--------+   |    +--------+     |
|  |        |         |      |        |   |    |        |     |
|  +--------+         |      +--------+<--|    +--------+     |
|  |        |         |      |        |-----|  |        |     |
|  +--------+<--------|      +--------+     |  +--------+     |
|  |  PDE   |----|           |   PTE  |     |  |  Data  |     |
|  +--------+    |           +--------+     |  +--------+<----|
|  |        |    |           |        |     |  |        |
|  +--------+    |           +--------+     |  +--------+
|  |        |    |           |        |     |  |        |
|->+--------+    |---------->+--------+     |->+--------+
     一级页表                    二级页表          物理内存页
```

这里我们再额外算一笔账，二级页表中每个表项占 32 位，所以一个一级页表的总体积是 `4byte x 1024 = 4KB`，而每个一级页表项都对应一个二级页表，所以全部二级页表的总体积是 `4KB x 1024 = 4MB`，即二级页表分页机制自身内存占用也要约 4MB 外加 4KB。

我们还要额外提一下页表项。上面刚说过每个页表项占 32 位，它也是分两个部分：高 20 位是基地址，低 12 位是控制标记位。所以当我们通过一级页表索引在一级页表中查找时是这样的：

```
一级页表项地址 = cr3寄存器高20位 + ( 10位一级页表索引 << 2 )
```

通过二级页表索引在二级页表中查找时是这样的：

```
二级页表项地址 = 一级页表项高20位 + ( 10位二级页表索引 << 2 )
```

读到这里你是否可以理解 `页表项索引左移 2 位` 这个操作的意义？索引就好比数组的下标，而这里我们要通过下标得到具体的位置，如果一条笔直的马路上每隔 2 米就插一面旗子，我现在站在这条路的起点处（20位基地址）问你第 5 面旗子（下标索引）距离我多远（地址），那么你一定会算：`5 x 2 = 10 米`，那么同理：

```
页表项地址 = 基地址 + ( 索引 x 页表项大小 )
           = 基地址 + ( 索引 x 4字节 )
           = 基地址 + ( 索引 << 2 )
```

最后我们再看看页表项低 12 位控制位都代表什么意义

```
+ 11 + 10 + 9 + 8 +  7 + 6 + 5 +  4  +  3  + 2  + 1  + 0 +
|    Avail    | G | PS | D | A | PCD | PWT | US | RW | P |
+--------------------------------------------------------+
|     000     | 0 |  1 | 0 | 0 |  0  |  0  |  0 |  1 | 1 |
+--------------------------------------------------------+

P     : 0 表示此页不在物理内存中，1 表示此页在物理内存中
RW    : 0 表示只读，1 表示可读可写（要配合 US 位）
US    : 0 表示特权级页面，1 表示普通权限页面
PWT   : 1 表示写这个页面时直接写入内存，0 表示先写到缓存中
PCD   : 1 表示该页禁用缓存机制，0 表示启用缓存
A     : 当该页被初始化时为 0，一但进行过读/写则置为 1
D     : 脏页标记（这里就不做具体介绍了）
PS    : 0 表示页面大小为 4KB，1 表示页面大小为 4MB
G     : 1 表示页面为共享页面（这里就不做具体介绍了）
Avail : 3 位保留位
```

然后我们再说回 xv6。

到目前为止我们知道 xv6 开启了 4MB 内存页大小，在 x86 架构下当通过 `cr4` 控制寄存器的 `PSE` 位打开了 4MB 分页后 MMU 内存管理单元的分页机制就会从二级分页简化位一级分页。

即虚拟地址的高 10 位仍然是一级页表项索引，但是后面的 22 位则全部变为页内偏移量（因为一页有 `2^22 = 4MB` 大小了嘛）。

我们来看看这个一级页表的结构

```
# Set page directory
  movl    $(V2P_WO(entrypgdir)), %eax
  movl    %eax, %cr3
```

通过代码我们知道页表地址是存在一个叫做 `entrypgdir` 的变量中了，通过文本搜索可以在 `main.c` 文件的最后找到这个变量，我们看一下

```
// Boot page table used in entry.S and entryother.S.
// Page directories (and page tables), must start on a page boundary,
// hence the "__aligned__" attribute.  
// Use PTE_PS in page directory entry to enable 4Mbyte pages.
__attribute__((__aligned__(PGSIZE)))
pde_t entrypgdir[NPDENTRIES] = {
  // Map VA's [0, 4MB) to PA's [0, 4MB)
  [0] = (0) | PTE_P | PTE_W | PTE_PS,
  // Map VA's [KERNBASE, KERNBASE+4MB) to PA's [0, 4MB)
  [KERNBASE>>PDXSHIFT] = (0) | PTE_P | PTE_W | PTE_PS,
};

//PAGEBREAK!
// Blank page.
```

将这些宏定义都转义过来我们看看这个页表的样子

```c
unsigned int entrypgdir[1024] = {
    [0] = 0 | 0x001 | 0x002 | 0x080,  // 0x083 = 0000 1000 0011
    [0x80000000 >> 22] = 0 | 0x001 | 0x002 | 0x080  // 0x083
};
```

可见这个页表非常简单，只有两个页表项 `0x00000000` 和 `0x80000000`，而且两个页表项索引的内存物理地址都是 `0 ~ 4MB`，其他页表项全部未作设置。而且通过这两个页表项的值也可以清楚的看出这段基地址为 0 的 4MB 大小的内存页还是特级权限内存页（低 12 位的控制位对应关系已经附在上面解释控制位的示意图里了）。

不难想象这么简单的页表肯定不是 xv6 最终使用的页表。这里可以先剧透一下，这确实只是一个临时页表，它只保证内核在即将打开内存分页支持后内核可以正常执行接下来的代码，而内核在紧接着执行 `main` 方法时会马上再次重新分配新的页表，而且最终的页表是 4KB 单位页面的精细页表哦~

###开启内存分页机制

我们在上上篇[《【学习xv6】从实模式到保护模式》](http://leenjewel.github.io/blog/2014/07/29/%5B%28xue-xi-xv6%29%5D-cong-shi-mo-shi-dao-bao-hu-mo-shi/)讲到如何打开保护模式时其实就已经介绍过开启分页机制的方法了：将 `cr0` 寄存器的第 31 位置为 1。

这里还要在提一句，至此我们开启了内存分页机制，接下来内核的代码执行和数据读写都要经过 MMU 通过分页机制对内核指令地址和数据地址的转换，那么目前的页表是如何保证在转换后的物理地址是正确的，如何保证内核可以继续正常运行的呢？

我们来分析一下。

根据 `kernel.ld` 链接器脚本的设定，内核的虚拟地址起始于 `0x80100000` 即内核代码段的起始处，而内核的代码段被放置在内存物理地址 `0x100000` 处。我们刚刚看到目前的临时页表将虚拟地址 `0x80000000` 映射到物理内存的 `0x0` 处，所以我们来尝试用刚刚了解到的内存分页机制来解析一下 `0x80100000` 虚拟地址最后转换成物理地址是多少。

```
0x80100000 = 1000 0000 00|01 0000 0000 0000 0000 0000

0x80100000 高 10 位 = 1000 0000 00 = 512

0x80100000 后 22 位 = 01 0000 0000 0000 0000 0000 = 1048576

索引 512 对应  entrypgdir[ 0x80000000 >> 22 ] 即基地址为 0x0

换算的物理地址 = 0 + 1048576 = 1048576 = 0x100000 

即内核代码段所在内存物理地址 0x100000
```

说白了就是通过页表将内核高端的虚拟地址直接映射到内核真实所在的低端物理内存位置。

这样虽然保证了在分页机制开启的情况下内核也可以正常运行，但也限制了内核最多只能使用 4MB 的内存，不过对于现在的内核来说 4MB 足够了。

###设置内核栈顶位置并跳转到 main 执行

```
  # Set up the stack pointer.
  movl $(stack + KSTACKSIZE), %esp

  # Jump to main(), and switch to executing at
  # high addresses. The indirect call is needed because
  # the assembler produces a PC-relative instruction
  # for a direct jump.
  mov $main, %eax
  jmp *%eax

.comm stack, KSTACKSIZE
```

这里通过 `.comm` 在内核 bbs 段开辟了一段 `KSTACKSIZE = 4096 = 4KB` 大小的内核栈并将栈顶设置为这段数据区域的末尾处（栈是自上而下的嘛），最后通过 `jmp` 语句跳转到 `main` 方法处继续执行。

看到 `main` 这个单词玩过 C 语言的会觉得亲切吧。没错，我们即将踏入 C 语言的天地了。顺带提一句，看过这篇之后你应该能想明白为什么 `main` 函数会是 C 语言编写的程序的入口（链接器脚本），可不可以用别的函数做 C 语言编写程序的入口呢？（可以，通过链接器脚本）。

##内核运行

我们来到 `main.c` 文件的 `main` 函数处，这里很干净的调用了一连串的方法。

```c
// Bootstrap processor starts running C code here.
// Allocate a real stack and switch to it, first
// doing some setup required for memory allocator to work.
int
main(void)
{
  kinit1(end, P2V(4*1024*1024)); // phys page allocator
  kvmalloc();      // kernel page table
  mpinit();        // collect info about this machine
  lapicinit();
  seginit();       // set up segments
  cprintf("\ncpu%d: starting xv6\n\n", cpu->id);
  picinit();       // interrupt controller
  ioapicinit();    // another interrupt controller
  consoleinit();   // I/O devices & their interrupts
  uartinit();      // serial port
  pinit();         // process table
  tvinit();        // trap vectors
  binit();         // buffer cache
  fileinit();      // file table
  iinit();         // inode cache
  ideinit();       // disk
  if(!ismp)
    timerinit();   // uniprocessor timer
  startothers();   // start other processors
  kinit2(P2V(4*1024*1024), P2V(PHYSTOP)); // must come after startothers()
  userinit();      // first user process
  // Finish setting up this processor in mpmain.
  mpmain();
}
```

至此我们已经了解一台 PC 从加电启动开始如何从实模式到保护模式、内存寻址如何从分段式到分页式，启动方式如何从 BIOS 到引导区程序再从引导区程序加载内核到内存中运行。

即便写了这么多，内核这位“神秘的少女”也只是刚刚走到我们面前，我们还未揭开这位“神秘的少女”的面纱去窥探她美丽的容貌。不过我们距离这一刻已经非常非常的近了，接下来我们将会看到 xv6 的内核是如何实现内存管理、进程管理、IO 操作等现代化操作系统所应该实现的诸多特性。

让我们继续加油！

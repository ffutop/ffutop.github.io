---
title: 理解 Linux Kernel (1) - BIOS
author: fangfeng
date: 2018-08-19
categories:
  - 技术
tags:
  - Linux
  - Kernel
  - BIOS
---

## 前言

在[概述](https://www.ffutop.com/2018-08-19-understand-Kernel-0/)，我已经介绍过《理解 Linux Kernel》系列文章的写作原因。我不能担保我所进行的所有试验性操作都是对的，但至少操作我的环境下成功地运行了，并帮助我触及我始终敬畏的**硬件&OS**

《Linux 内核完全注释》第三章——内核编程语言和环境，描述了用 as86 汇编语言构建 boot 引导程序，在 Bochs 仿真器成功模拟开机运行，最终输出 *Loading System...*。这就是本篇所要尝试的核心实验。之所以在已经有资料的基础上再写一遍，是书中缺失了仿真器模拟的环节。

<!--more-->

## Bochs 仿真器

[Bochs 官网](https://bochs.sourceforge.net/)

[Bochs 下载链](https://sourceforge.net/projects/bochs/files/)

用 Bochs 仿真器跑 0.11 版本内核绝对是一个明智的选择，实践出真知放在这里绝对不会错。但说实话这一周都差不多要被 Bochs 折腾死，用源码进行编译出现各种各样的问题。(我猜新版本根本就没考虑对 macOS 做兼容)

最后，不得不使用经编译后的二进制分发版。 `brew install bochs` 当然了，反正也能用，就不纠结非得自己编译了。(如果哪位能成功编译个 macOS 可用的应用，希望能把踩得坑整理整理，供后来者参考)

### 目录结构

```
.
├── CHANGES
├── COPYING
├── INSTALL_RECEIPT.json
├── LICENSE
├── README
├── TODO
├── bin
│   ├── bochs                           // bochs 可执行文件
│   └── bximage                         // 制作磁盘映像文件的工具
├── lib                                 // 动态库目录
│   └── bochs
│       └── plugins
│           ├── libbx_acpi.0.0.0.so
│           ├── ... 略
│           └── libbx_vga.so -> libbx_vga.0.0.0.so
└── share                               // 与体系结构无关的文件放在此目录下
    ├── bochs
    │   ├── BIOS-bochs-latest
    │   ├── BIOS-bochs-legacy
    │   ├── SeaBIOS-README
    │   ├── VGABIOS-elpin-2.40
    │   ├── VGABIOS-elpin-LICENSE
    │   ├── VGABIOS-lgpl-README
    │   ├── VGABIOS-lgpl-latest
    │   ├── VGABIOS-lgpl-latest-cirrus
    │   ├── VGABIOS-lgpl-latest-cirrus-debug
    │   ├── VGABIOS-lgpl-latest-debug
    │   ├── bios.bin-1.7.5
    │   └── keymaps
    │       ├── sdl-pc-us.map
    │       ├── ... 略
    │       └── x11-pc-us.map
    ├── doc
    │   └── bochs
    │       ├── CHANGES
    │       ├── COPYING
    │       ├── LICENSE
    │       ├── README
    │       ├── TODO
    │       ├── bochsrc-sample.txt      // bochsrc 配置文件的示例样板
    │       └── slirp.conf
    └── man
        ├── man1
        │   ├── bochs-dlx.1.gz
        │   ├── bochs.1.gz
        │   └── bximage.1.gz
        └── man5
            └── bochsrc.5.gz
```

### 简单使用

Bochs 下载链提供了 Bochs 不同版本的源码的下载，同时也提供了一些可用的磁盘映像文件(都是一些已经经过处理，可用使用进行仿真运行的 OS )

说实话，这是绝对有必要进行的一步，不仅仅是了解配置文件 bochsrc 的配置方式，更可用借此了解一下仿真器的使用方式。(在启动系统上，我也被坑了一天，后面细说)

首先，这里演示的将是 DLX Linux 。

1. 下载，解压，进入目录。

2. 目录下文件如下

```plain
.
├── bochsrc.txt         // bochs 启动时将默认通过这个文件读取仿真器配置
├── hd10meg.img         // 磁盘映像文件
├── readme.txt
└── testform.txt
```

3. 启动仿真器

在 DLX Linux 目录下键入命令 `bochs`, 观察到命令行输出:

```plain
========================================================================
                       Bochs x86 Emulator 2.6.9
               Built from SVN snapshot on April 9, 2017
                  Compiled on May  2 2018 at 13:26:32
========================================================================
00000000000i[      ] LTDL_LIBRARY_PATH not set. using compile time default '/usr/local/Cellar/bochs/2.6.9_2/lib/bochs/plugins'
00000000000i[      ] BXSHARE not set. using compile time default '/usr/local/Cellar/bochs/2.6.9_2/share/bochs'
00000000000i[      ] lt_dlhandle is 0x7fef2252a500
00000000000i[PLUGIN] loaded plugin libbx_usb_common.so
00000000000i[      ] lt_dlhandle is 0x7fef2252a8a0
00000000000i[PLUGIN] loaded plugin libbx_unmapped.so
00000000000i[      ] lt_dlhandle is 0x7fef2252ad00
00000000000i[PLUGIN] loaded plugin libbx_biosdev.so
00000000000i[      ] lt_dlhandle is 0x7fef2252b1e0
00000000000i[PLUGIN] loaded plugin libbx_speaker.so
00000000000i[      ] lt_dlhandle is 0x7fef2252ba50
00000000000i[PLUGIN] loaded plugin libbx_extfpuirq.so
00000000000i[      ] lt_dlhandle is 0x7fef2252be70
00000000000i[PLUGIN] loaded plugin libbx_parallel.so
00000000000i[      ] lt_dlhandle is 0x7fef238013c0
00000000000i[PLUGIN] loaded plugin libbx_serial.so
00000000000i[      ] lt_dlhandle is 0x7fef22706cd0
00000000000i[PLUGIN] loaded plugin libbx_iodebug.so
00000000000i[      ] reading configuration from bochsrc.txt
------------------------------
Bochs Configuration: Main Menu
------------------------------

This is the Bochs Configuration Interface, where you can describe the
machine that you want to simulate.  Bochs has already searched for a
configuration file (typically called bochsrc.txt) and loaded it if it
could be found.  When you are satisfied with the configuration, go
ahead and start the simulation.

You can also start bochs with the -q option to skip these menus.

1. Restore factory default configuration
2. Read options from...
3. Edit options
4. Save options to...
5. Restore the Bochs state from...
6. Begin simulation
7. Quit now

Please choose one: [6]
```

bochs 默认读取当前目录下 `bochsrc.txt` 文件，因此不需要其他配置。

选择 6 或者直接 *回车*

```plain
00000000000i[      ] lt_dlhandle is 0x7fef22707270
00000000000i[PLUGIN] loaded plugin libbx_sdl2.so
00000000000i[      ] installing sdl2 module as the Bochs GUI
00000000000i[SDL2  ] maximum host resolution: x=2880 y=1800
00000000000i[      ] using log file bochsout.txt
Next at t=0
(0) [0x0000fffffff0] f000:fff0 (unk. ctxt): jmpf 0xf000:e05b          ; ea5be000f0
<bochs:1>
```

![](https://ws1.sinaimg.cn/large/006tNbRwgy1fuevtktuh0j30zc0tcwfd.jpg)

终端出现更多输出，同时 Bochs 开启了一个可视化终端(当然，现在还没有任何内容)。

在这儿被坑了好久。本来以为 Bochs 会直接执行 BIOS 指令，启动操作系统，但是自始至终，停在这里就没动过...

这边是由于 Bochs 本身是支持调试的(T\_T，见鬼)，就像是 GDB 和 LLDB 一样，进入调试后，需要指令 `c` (continue) 来继续执行(当然，还有其它调试命令)

4. 键入 `c` 

```plain
<bochs:1> c
```

仿真器开始引导程序的加载

![](https://ws4.sinaimg.cn/large/006tNbRwgy1fuew9zoocuj313c0pstaw.jpg)

OS Kernel 启动，等待用户登录 (默认的用户是 root , 没有密码)。
之后就和普通的 Linux 机器类似，不过支持的命令比较少。毕竟是一个精简后的系统。

![](https://ws4.sinaimg.cn/large/006tNbRwgy1fuewamqzm6j313q0pqjtw.jpg)

5. 关机

![](https://ws3.sinaimg.cn/large/006tNbRwgy1fuewi9wjawj313q0q2gnc.jpg)

仿真器仿真了许多实体机案件，右上角最后一个就是关机键

### Docker 容器中运行 Ubuntu

这部分不详述，作为一个热门的技术，这方面的资料很多。Ubuntu 系统只是为了来执行一些 macOS 上没有的命令的(比如 `as86`, `ld86`)

简述两个 docker 容器和宿主机间复制文件的命令 

在宿主机下执行命令:

```sh
docker cp [OPTIONS] CONTAINER:SRC_PATH DEST_PATH|-
docker cp [OPTIONS] SRC_PATH|- CONTAINER:DEST_PATH [flags]
```

例如:

```sh
docker cp linux:/root/hello.c ./hello.c         # 这里 linux 是 docker 容器名
docker cp 92dfc8ad70e1:/root/hello.c ./hello.c  # 这里 92dfc8ad70e1 是 docker 容器 ID
```


## boot.s 汇编程序

这里没法多说，都是书中的源码，只是供未读过书的读者参考，特意摘录了下来(不过，就不加注释了)

boot.s 源代码

```asm
.global begtext, begdata, begbss, endtext, enddata, endbss
.text
begtext:
.data
begdata:
.bss
begbss:
.text
BOOTSEG=0x07c0

entry start
start:
    jmpi    go,BOOTSEG
go: mov ax,cs
    mov ds,ax
    mov es,ax
    mov [msg+17],ah
    mov cx,#20
    mov dx,#0x1004
    mov bx,#0x000c
    mov bp,#msg
    mov ax,#0x1301
    int 0x10
loop1:  jmp loop1
msg:    .ascii  "Loading System..."
        .byte   13,10
.org    510
.word   0xAA55
.text
endtext:
.data
enddata:
.bss
endbss:
```

这段程序的主要的执行流程将是:

1. 通过 BIOS 加载这段 boot 引导程序
2. 红色字体打印 *Loading System...* 并响铃
3. 指令自循环 (`loop1 jmp loop1`) ，将始终展示上述字样，并不接收命令

下面就该把这个汇编程序 *编译 + 链接* 成 boot 引导程序。

虽然在 macOS 上确实找到了一个 as86 汇编器(来自 nasm 的一个编译选项，但我不知道为什么始终无法正常编译，也可能这个根本就不是我想要的)。

通过 docker 容器部署的 Ubuntu，可以很容易地得到一个 as86 汇编程序。

```sh
apt-get install bin86   # as86, ld86 都在这个包里提供了

# 这句需要在宿主机上执行
docker cp boot.s linux:/root/boot.s     # 这里 linux 是我 docker 部署的 Ubuntu 的容器的名称。   如果之前 boot.s 在宿主机上，可以这样拷贝到容器中

as86 -0 -a -o boot.o boot.s             # 编译

ld86 -0 -s -o execfile boot.o           # 链接

# 当然，到这里为止，其实都可以理解成在将汇编代码编译链接成可执行程序 (和 boot 引导程序没有太大关系，唯一有关系的就是这段汇编语言是以引导为目的写的)

dd bs=32 if=execfile of=boot skip=1     # 这里去掉可执行程序的前 32 字节，形成刚好 512 字节的 boot 引导程序
```

## 用仿真器启动引导程序

事实上，这部分内容，我始终没有搞清楚 **磁盘映像文件** 和 **boot 引导程序** 间的关系(当然还有 floppy 和 ata0~3)

在上一节成功拿到 *512B* 的 boot 引导程序之后，直接使用这个貌似我也运行成功了(根本就不需要创建磁盘映像文件并将引导程序写入映像文件中，不过，也许只是因为这个引导程序太简单了，根本就不需要其他程序的配合。它最后就是一个死循环 )

总之，先按照最简单的来吧。

把 boot 引导程序拷贝到宿主机上(貌似频繁地在两个OS上交互文件，这个很无奈，docker容器中的进程难以启动显示程序)

1. 在宿主机新建一个目录 *linux-boot*
2. 拷贝 boot 引导程序到宿主机上 `docker cp linux:/root/boot linux-boot/`
3. 在 *linux-boot* 下建立 bochsrc 文件(这将是整个仿真器的配置，用于模拟组装机器涉及的 CPU, 内存, BIOS, 显示器等)
4. 这里使用的配置文件如下:

```rc
# You may now use double quotes around pathnames, in case
# your pathname includes spaces.

cpu: model=pentium, count=1, ips=50000000, reset_on_triple_fault=1, ignore_bad_msrs=1, msrs="msrs.def"
cpu: cpuid_limit_winnt=0

memory: guest=512, host=256

romimage: file=$BXSHARE/BIOS-bochs-legacy, options=fastboot     # BIOS 配置

vgaromimage: file=$BXSHARE/VGABIOS-lgpl-latest

mouse: enabled=0

pci: enabled=1, chipset=i440fx

private_colormap: enabled=0

floppya: 1\_44="./boot", status=inserted         # 装载一个软盘 A ，这里用当前目录下的 boot (boot 引导程序，事实上书中说要用 .img 磁盘映像文件，但这里不影响实际效果)

ata0: enabled=1, ioaddr1=0x1f0, ioaddr2=0x3f0, irq=14
ata1: enabled=1, ioaddr1=0x170, ioaddr2=0x370, irq=15
ata2: enabled=0, ioaddr1=0x1e8, ioaddr2=0x3e0, irq=11
ata3: enabled=0, ioaddr1=0x168, ioaddr2=0x360, irq=9

boot: a                                         # 配置引导程序所在的磁盘

floppy_bootsig_check: disabled=0

log: bochsout.txt

panic: action=ask
error: action=report
info: action=report
debug: action=ignore, pci=report # report BX_DEBUG from module 'pci'

debugger_log: -

parport1: enabled=1, file="parport.out"

speaker: enabled=1, mode=sound
```

5. 当前目录 *linux-boot* 下，键入命令 `bochs`
6. 由于读取到 *bochsrc* 配置文件的存在，menu 的默认选项为 6 (开始模拟机器)

```plain
You can also start bochs with the -q option to skip these menus.

1. Restore factory default configuration
2. Read options from...
3. Edit options
4. Save options to...
5. Restore the Bochs state from...
6. Begin simulation
7. Quit now

Please choose one: [6]
```

7. 直接开始运行机器，键入命令 `c`

```plain
Please choose one: [6]
00000000000i[      ] lt_dlhandle is 0x7f85d0405ff0
00000000000i[PLUGIN] loaded plugin libbx_sdl2.so
00000000000i[      ] installing sdl2 module as the Bochs GUI
00000000000i[SDL2  ] maximum host resolution: x=2880 y=1800
00000000000i[      ] using log file bochsout.txt
Next at t=0
(0) [0x0000fffffff0] f000:fff0 (unk. ctxt): jmpf 0xf000:e05b          ; ea5be000f0
<bochs:1> c
```

8. 观察仿真器的表现

![](https://ws4.sinaimg.cn/large/006tNbRwgy1fuf2kicsqyj313y0q0gn4.jpg)

Oh, YES! 成功输出了 *Loading System...* (不过响铃没有听到，可能与我没有配置 sound 有关)

9. 关机

无需多言，右上角模拟的就是**关机实体按键**

```plain
  __                    __                  
 / _| __ _ _ __   __ _ / _| ___ _ __   __ _ 
| |_ / _` | '_ \ / _` | |_ / _ \ '_ \ / _` |
|  _| (_| | | | | (_| |  _|  __/ | | | (_| |
|_|  \__,_|_| |_|\__, |_|  \___|_| |_|\__, |
                 |___/                |___/ 
```

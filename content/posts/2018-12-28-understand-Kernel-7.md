---
title: 理解 Linux Kernel (7) - 字符设备
author: fangfeng
date: 2018-12-28
categories:
  - 技术
tags:
  - Linux
  - Kernel
  - Char Dev
---

相比较于块设备，字符设备无论从物理认知上，抑或是理论理解上，都存在着相当大的入门门槛。特别是在将字符设备与控制台、命令行终端混淆的时候，就更加难以进行分辨了。

回到字符设备本身，字符设备与块设备最主要的区别就在于块设备可以随机读写，而字符设备只能够顺序读，顺序写。

那么，常见的字符设备有什么？显示器、键盘、鼠标。

<!--more-->

## 宏观概览

通常在我们的认识中，命令行终端就被认为是与一套字符设备相配合来使用的。很正常嘛。我们打开一个 Shell ，通过键盘输入一些字符，配合显示器把这些经过加工的字符展示出来。

那么，是不是就意味着 Shell 作为一个任务(进程)，以键盘设备作为标准输入，以显示设备作为标准输出?

看看一个 1 号任务 `/bin/bash` 的文件描述符说明吧。

```sh
$ ls -al
total 0
dr-x------ 2 root root  0 Dec 13 23:20 .
dr-xr-xr-x 9 root root  0 Dec 13 23:20 ..
lrwx------ 1 root root 64 Dec 13 23:20 0 -> /dev/pts/0
lrwx------ 1 root root 64 Dec 13 23:20 1 -> /dev/pts/0
lrwx------ 1 root root 64 Dec 13 23:20 2 -> /dev/pts/0
lrwx------ 1 root root 64 Dec 25 01:09 255 -> /dev/pts/0
```

这里文件描述符 0, 1, 2 分别表示任务的标准输入、标准输出、标准错误。这些内容在表现形式上做了一个链接。

那么，`/dev/pts/0` 是什么? 

```sh
crw--w---- 1 root tty 136, 0 Dec 13 23:20 /dev/pts/0
```

一个字符设备。到此为止，似乎和最初预想的有比较大的区别。至少在假象中，至少是一个输入(键盘)，一个输出(显示器)。怎么就混成了一个呢？

本来是怎么都想不通的，但后来配合"Unix一切皆文件"的信条，总算是有点明白了。

相信大家都有这这样的经历，某个程序是以键盘输入作为标准输入的。其实就和上面👆展示的一样 `0 -> /dev/pts/0` 。那么，有没有考虑过这整套流程是怎么协作的呢？

![](https://ws4.sinaimg.cn/large/006tNbRwly1fyjvtjiu0zj31980oqmxe.jpg)

对于程序来说，我们还是普通的调用 `read`, `write` 等经过封装的函数，来读取一个所谓的文件。

但对于文件是字符设备时，最终调用的就是 `tty_read`, `tty_write` 了。通俗的讲，大概率的就是键盘作为输入，显示器作为输出了。

## 源码剖析

### 文件读写

这部分上一篇已经介绍过了，不做过多说明。

简单回顾下 `sys_read` 函数

```c
int sys_read(unsigned int fd,char * buf,int count)
{
	struct file * file;
	struct m_inode * inode;

	if (fd>=NR_OPEN || count<0 || !(file=current->filp[fd]))
		return -EINVAL;
	if (!count)
		return 0;
	verify_area(buf,count);
	inode = file->f_inode;
	if (inode->i_pipe)
		return (file->f_mode&1)?read_pipe(inode,buf,count):-EIO;
    /* 确认到i节点描述的是字符设备 */
	if (S_ISCHR(inode->i_mode)) 
		return rw_char(READ,inode->i_zone[0],buf,count,&file->f_pos);
	if (S_ISBLK(inode->i_mode))
		return block_read(inode->i_zone[0],&file->f_pos,buf,count);
	if (S_ISDIR(inode->i_mode) || S_ISREG(inode->i_mode)) {
		if (count+file->f_pos > inode->i_size)
			count = inode->i_size - file->f_pos;
		if (count<=0)
			return 0;
		return file_read(inode,file,buf,count);
	}
	printk("(Read)inode->i_mode=%06o\n\r",inode->i_mode);
	return -EINVAL;
}
```

可以看到 `S_ISCHR()` 就是在对i节点的类型进行判别，从而进行不同的分发。

```c
static crw_ptr crw_table[]={
	NULL,		/* nodev */
	rw_memory,	/* /dev/mem etc */
	NULL,		/* /dev/fd */
	NULL,		/* /dev/hd */
	rw_ttyx,	/* /dev/ttyx */
	rw_tty,		/* /dev/tty */
	NULL,		/* /dev/lp */
	NULL};		/* unnamed pipes */
/**
 * 这段还是涉及到分发，由不同的设备号(dev) 来确定执行函数 (ttyx, 串口终端; tty, 控制台终端; mem, /dev/mem 等)
 * crw_ptr 是 C 语言中常见的函数指针。由dev号来确定调用哪个函数
 */
int rw_char(int rw,int dev, char * buf, int count, off_t * pos)
{
	crw_ptr call_addr;

	if (MAJOR(dev)>=NRDEVS)
		return -ENODEV;
	if (!(call_addr=crw_table[MAJOR(dev)]))
		return -ENODEV;
	return call_addr(rw,MINOR(dev),buf,count,pos);
}
```

执行到 `rw_tty`, `rw_ttyx` 两个函数，就将对读/写进行区分，并由特定的函数进行处理。

```c
static int rw_ttyx(int rw,unsigned minor,char * buf,int count,off_t * pos)
{
	return ((rw==READ)?tty_read(minor,buf,count):
		tty_write(minor,buf,count));
}

static int rw_tty(int rw,unsigned minor,char * buf,int count, off_t * pos)
{
	if (current->tty<0)
		return -EPERM;
	return rw_ttyx(rw,current->tty,buf,count,pos);
}
```

到此为止，应该能够基本了解了字符设备，作为 Linux 的文件，所应该有的读写的相关支持。

但是，究竟读、写的内容在哪呢？比如正规文件，都很容易地可以理解到，就是从磁盘中取出某个/某几个盘块的内容即可。字符设备的IO要从哪里取(往哪里送)数据呢?

### 字符设备驱动

对于设备驱动这个概念，至今没有搞清楚。不过，这不妨碍对代码的理解。反正对于字符设备驱动来说，也就是 Linux 内核中的一些软件层面的代码，与普通代码的唯一区别，就是对外设硬件做了相应的交互支持。

![](https://ws4.sinaimg.cn/large/006tNbRwly1fykyc6lo9pj31i40u0ac8.jpg)

承接操作系统的字符设备接口，`tty_read`、`tty_write` 负责读入和写出。

从哪里读？`secondary` 数据队列；往哪里写？`write_q` 数据队列。

同时，也可以看到，与这些队列直接相关的，就是我们熟知的键盘和显示器了。是不是有了点豁然开朗的感觉。

好，我们先来看看这里描述的三个队列究竟是怎么工作的。这里就不得不先看看内存中抽象出来的描述终端的数据结构 `tty_struct`。

```c
/**
 * copied from include/linux/tty.h
 */
struct tty_struct {
	struct termios termios;     /* terminal IO conf */
	int pgrp;                   /* 所属进程组 */
	int stopped;                /* 停止标志 */
	void (*write)(struct tty_struct * tty); /* 终端写函数指针 */
	struct tty_queue read_q;    /* 终端读队列 */
	struct tty_queue write_q;   /* 终端写队列 */
	struct tty_queue secondary; /* 终端辅助队列 */
};

/**
 * copied from include/termios.h
 */
struct termios {                /* terminal IO 属性 */
	unsigned long c_iflag;		/* input mode flags */
	unsigned long c_oflag;		/* output mode flags */
	unsigned long c_cflag;		/* control mode flags */
	unsigned long c_lflag;		/* local mode flags */
	unsigned char c_line;		/* line discipline */
	unsigned char c_cc[NCCS];	/* control characters */
};

/**
 * copied from include/linux/tty.h
 */
struct tty_queue {
	unsigned long data;         /* 字符行数量 | 串口终端则存储端口号 */
	unsigned long head;         /* 头指针 */
	unsigned long tail;         /* 尾指针 */
	struct task_struct * proc_list; /* 等待该终端的任务队列 */
	char buf[TTY_BUF_SIZE];     /* 队列的缓冲区 */
};
```

对于任何任务需要读写字符设备(这里指终端设备)，最终直接读取/写入到的就是 `secondary` 和 `write_q`。

---

这里可能有个小小的疑问? 为什么读终端设备不是读 `read_q` 呢？

其实也比较好解释，相信日常在操作命令行的时候，我们在键盘上敲击的键位与显示器上实际展示的内容是不相匹配的。最普通的，我们键入了 **delete(删除键)**，为什么不是一个 **delete** 对应的 ascii (当然，这里请先忽视非打印字符的问题)，而是删除了最后一个字符呢？完全可以想象，反正任何键位与驱动交互的时候都是传入一串二进制码嘛。

这里的 `secondary` 完成的就是怎么一个工作，`read_q` 存储的是所有的外设(这里是键盘)的输入，并原样存储。而到了 `secondary` 就是经过相应的加工，诸如控制字符之类的都展现了各自的意义，并完成一些加工工作，而不再仅仅只是普通地展示了。

---

另外，可能还有一个问题。虽然我们已经习惯了在命令行交互时，所有的输入都将直接在显示器上进行展示，好像我们在直接往显示器上写内容。但是，前面我们描述的内容都是，键盘设备作为输入，是要写到 `read_q` 乃至 `secondary` 并最终作为进程的输入的。在显示器上显示并不是它本来应该干的事情 (比较显示器上展示的应该是 `write_q` 的内容，也就是进程的标准输出)

事实上，这仅仅只是一个回显，将 `secondary` 队列的内容复制了一份到写队列，也就呈现出让显示器打印相应键盘输入的效果了。

同时，这也就能够直接解释为什么我们在使用 `passwd`, `su`, `sudo` 等命令时，要求输入密码都是不在显示器上回显的。实现也相当简单了嘛。把 tty_struct.termios 的相应控制属性 (ECHO) 重置，就可以实现不回显的效果了。

### 终端设备交互

最后一部分，应该也是最关心的了。外设如何与操作系统完成交互。其实也能够想得到了——中断。

在操作系统初始化时，就把中断描述符表的中断表项配置得当。之后的事情就是等待键盘等外设输入的中断信号了。

又看回到了 `init/main.c` 程序

```c
void main(void)
{
    ...
	trap_init();
	blk_dev_init();
	chr_dev_init(); /* 块设备相关初始化, 方法体是空的，没有实现 */
	tty_init();     /* tty 终端设备初始化 */
	time_init();
	sched_init();
	buffer_init(buffer_memory_end);
    ...
}
```

```
/**
 * Copied from kernel/chr_drv/tty_io.c
 * 终端设备初识化
 */
void tty_init(void)
{
    /** 串口设备初始化 */
	rs_init();
    /** 控制台设备初始化 */
	con_init();
}
```

至于为什么有两种初始化方式。这来源于终端又区分为控制台终端与串口终端，区别就是一个是直接建立在主机上，串口终端是通过串行接口连接到主机的。当然，这都是古老的方式了，细节就不太清楚了。

下面来看看 `con_init()` 做了哪些工作(`rs_init()` 的内容请自行了解)

```c
void con_init(void)
{
	register unsigned char a;
	char *display_desc = "????";
	char *display_ptr;

    /**
     * 读取 setup.s 程序预处理的内容
     * 包括显示器的各种配置参数
     */
	video_num_columns = ORIG_VIDEO_COLS;
	video_size_row = video_num_columns * 2;
	video_num_lines = ORIG_VIDEO_LINES;
	video_page = ORIG_VIDEO_PAGE;
	video_erase_char = 0x0720;

    /**
     * 读取显示器的配置并进行相关设置 (省略代码)
     */
    ...

	origin	= video_mem_start;
	scr_end	= video_mem_start + video_num_lines * video_size_row;
	top	= 0;
	bottom	= video_num_lines;

	gotoxy(ORIG_X,ORIG_Y);
    /** 设置陷阱门 */
	set_trap_gate(0x21,&keyboard_interrupt);
	outb_p(inb_p(0x21)&0xfd,0x21);
	a=inb_p(0x61);
	outb_p(a|0x80,0x61);
	outb(a,0x61);
}
```

应该能够看到最重要的内容就是**设置键盘中断陷阱门**了。

之后只有静静地等待键位敲击，也就能够产生硬件中断，从而让 `read_q` 获得到相应的字符输入。

至于键盘中断的相应处理流程，这里不再详述。简述一些步骤:

1. 产生硬中断 `keyboard_interrupt`，由程序 `Keyboard.s` 的汇编代码进行处理

2. 根据不同的键盘(US 键盘、德式键盘等等)将获得的键位信号进行相应的字符转换(查转换表)

3. 调用 `do_tty_interrupt` 处理函数 (确认是给哪个终端的信号)

4. 调用 `copy_to_cooked(tty)` ，即完成 `read_q` 到 `secondary` 的相关加工。



```plain
  __                    __                  
 / _| __ _ _ __   __ _ / _| ___ _ __   __ _ 
| |_ / _` | '_ \ / _` | |_ / _ \ '_ \ / _` |
|  _| (_| | | | | (_| |  _|  __/ | | | (_| |
|_|  \__,_|_| |_|\__, |_|  \___|_| |_|\__, |
                 |___/                |___/ 
```

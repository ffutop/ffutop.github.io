---
title: 理解 Linux Kernel (4) - 任务调度
date: 2018-10-12
author: fangfeng
categories:
  - 技术
tags:
  - Kernel
  - Linux
  - time interrupt
---

前面几节已经描述过，对于单核 CPU 来说。CPU 就处于不断地执行指令的过程(或者通过 `hlt` 指令直接停止工作)。

针对于每一个程序来说，这个程序执行流程是通过 CPU 中几组寄存器(通用寄存器、段寄存器、控制寄存器等) 和存储在内存中的代码和数据协作完成的。

如果要达到单核多任务的目的，首先要做的就是完成对几组寄存器中当前值的保存(我称之为保存现场)。而对于内存来说，多个任务的代码、数据同时存在内存是完全合理且可行的。毕竟相较于有限的寄存器，内存实在是太大了(相对而言)。

<!--more-->

## 任务调度的宏观描述

从宏观上来说，操作系统维护了若干个任务(假设有 0, 1, 5, 6)。

下面以一个假象的例子来对任务调度做一些形象的说明:

1. 假设当前任务是任务 5 ，操作系统分配给它的 CPU 使用时间是 30ms。

2. 每 10ms 计时器(Intel 8253, 可编程计数器/定时器) 向 CPU 发起一个时钟中断。

3. CPU 开始处理时钟中断(此时是**内核态**)。当前任务剩余可用时间 -10ms。

4. 检查当前任务的剩余时间片，有剩余 -> 步骤 5 ; 否则 -> 步骤 6 

5. 直接退出对时钟中断的处理函数(回到 **用户态**), 重复步骤 1 

6. 根据每个任务的优先级及其它一些因素确定把接下来的一段时间分配给哪个任务。(假设分配给任务 6 ，30ms 的 CPU 使用时间) -> 重复步骤 2 。

当然，上面的描述中忽略了很多的因素。

## 任务调度准备阶段

这里都将以 Linux 0.11 版本代码作为实例。当然，其中一些代码为了适应现在的 GCC 做了一些修改，另外还可能直接摆出 GCC 编译后的汇编代码来解释说明。

首先，内核代码在经过一系列的准备流程(包括设置一些寄存器以及在内存中暂存某些值，读取计算机的硬件配置等)。真正开始出现任务 0 始于下列这段代码:

```c
/* 来自 Linux0.11 init/main.c */
void main(void)	
{
    ...

	sched_init();       /* schedule, 顾名思义就是时间安排咯 */
    
    ...
	sti();
	move_to_user_mode();
	if (!fork()) {	
		init();
	}
	for(;;) pause();
}
```

在操作系统的主函数中，开始对任务调度进行一定的准备工作。看看具体代码吧

```c
/* 这里给出的是 Linux0.11 kernel/sched.c 经过预编译的函数(里面有一些内联汇编) */
void sched_init(void)
{
    int i;
    struct desc_struct * p;    /* 声明一个描述符指针 */

    if (sizeof(struct sigaction) != 16)
        panic("Struct sigaction MUST be 16 bytes");
    __asm__ (                           /* 这段是为了设置全局描述符表GDT的第4项，是一个 0 号任务(当前任务)的任务调用门*/
            "movw $104,%1\n\t" 
            "movw %%ax,%2\n\t" 
            "rorl $16,%%eax\n\t" 
            "movb %%al,%3\n\t" 
            "movb $" "0x89" ",%4\n\t" 
            "movb $0x00,%5\n\t" 
            "movb %%ah,%6\n\t" 
            "rorl $16,%%eax" 
            ::"a" (&(init_task.task.tss)), "m" (*(((char *) (gdt+4)))), "m" (*(((char *) (gdt+4))+2)), "m" (*(((char *) (gdt+4))+4)), "m" (*(((char *) (gdt+4))+5)), "m" (*(((char *) (gdt+4))+6)), "m" (*(((char *) (gdt+4))+7)) 
            );
    __asm__ (                           /* 这里设置全局描述符表的第5项，0号任务的局部描述符表基址选择符 */
            "movw $104,%1\n\t" 
            "movw %%ax,%2\n\t" 
            "rorl $16,%%eax\n\t" 
            "movb %%al,%3\n\t" 
            "movb $" "0x82" ",%4\n\t" 
            "movb $0x00,%5\n\t" 
            "movb %%ah,%6\n\t" 
            "rorl $16,%%eax" 
            ::"a" (&(init_task.task.ldt)), "m" (*(((char *) (gdt+(4 +1))))), "m" (*(((char *) (gdt+(4 +1)))+2)), "m" (*(((char *) (gdt+(4 +1)))+4)), "m" (*(((char *) (gdt+(4 +1)))+5)), "m" (*(((char *) (gdt+(4 +1)))+6)), "m" (*(((char *) (gdt+(4 +1)))+7)) 
            );
    p = gdt+2+4;        /* 描述符指针指向 GDT 第6项。因为前面已经占用了第4，5项。由内核占用了 0，1，2，3。*/
    for(i=1;i<64;i++) {         /* 循环 63 次，这是 Linux0.11 最大支持 64 个任务同时存在。当前任务已经占了一个任务了*/
        task[i] = ((void *) 0); /* 63个任务指针全部值为 NULL */
        p->a=p->b=0;            /* GDT 累积 126 项(126 * 8 字节)全部置为 0 。每个任务占用 GDT 两项，一为任务门，一为局部描述符表选择符*/
        p++;
        p->a=p->b=0;
        p++;
    }

    __asm__("pushfl ; andl $0xffffbfff,(%esp) ; popfl");
    __asm__("ltr %%ax"::"a" (((((unsigned long) 0)<<4)+(4<<3))));           /* load Task Register TR 记录当前任务门为 gdt 第4项 */
    __asm__("lldt %%ax"::"a" (((((unsigned long) 0)<<4)+((4 +1)<<3))));     /* load Local Descriptor Table Register LDTR 记录当前选择符为 gdt 第5项 */
    /* 给8253芯片编程，每 10ms 发起一次时钟中断(下面3行的工作) */
    __asm__ ("outb %%al,%%dx\n" "\tjmp 1f\n" "1:\tjmp 1f\n" "1:"::"a" (0x36),"d" (0x43));   
    __asm__ ("outb %%al,%%dx\n" "\tjmp 1f\n" "1:\tjmp 1f\n" "1:"::"a" ((1193180/100) & 0xff),"d" (0x40));
    __asm__ ("outb %%al,%%dx"::"a" ((1193180/100) >> 8),"d" (0x40));
    __asm__ ("movw %%dx,%%ax\n\t" "movw %0,%%dx\n\t" "movl %%eax,%1\n\t" "movl %%edx,%2" : : "i" ((short) (0x8000+(0<<13)+(14<<8))), "o" (*((char *) (&idt[0x20]))), "o" (*(4+(char *) (&idt[0x20]))), "d" ((char *) (&timer_interrupt)),"a" (0x00080000)); /* 在 IDT 中设置时钟中断描述符项，第32项为时钟中断*/
    __asm__ ("outb %%al,%%dx"::"a" (({ unsigned char _v; __asm__ volatile ("inb %%dx,%%al\n" "\tjmp 1f\n" "1:\tjmp 1f\n" "1:":"=a" (_v):"d" (0x21)); _v; })&~0x01),"d" (0x21)); /*重新设置时钟中断的可屏蔽属性，这样就可在调用 hlt 后被时钟中断唤醒 */
    __asm__ ("movw %%dx,%%ax\n\t" "movw %0,%%dx\n\t" "movl %%eax,%1\n\t" "movl %%edx,%2" : : "i" ((short) (0x8000+(3<<13)+(15<<8))), "o" (*((char *) (&idt[0x80]))), "o" (*(4+(char *) (&idt[0x80]))), "d" ((char *) (&system_call)),"a" (0x00080000)); /* 在 IDT 中设置系统中断，第128项为时钟中断 */
}
```

通过执行 `sched_init()` ，操作系统开始有了任务的概念。并且把当前任务作为任务 0 来加以认识。

同时，预置了 64 个任务状态数组，用来辅助任务调度器完成未来调度任务时确认目前所有任务的基础性支持。

最重要的，当然是对 8253 芯片的编程，使其每 10ms 向 CPU 发起一个硬件时钟中断。由此将把系统暂时性的带入**内核态** 来完成 CPU 下一个 10ms 需要进行的任务的决策工作。

最后的设置时钟中断描述符和系统中断描述符自不用说。不设置的话，对于接收到的中断根本就没办法确认处理中断的代码在哪(毕竟中断处理逻辑也是由 CPU 执行指令来解决的)

## 任务调度实施阶段

由于时钟中断是由硬件芯片进行控制，根本不会顾及当前 CPU 正在执行的任务已经执行到了哪个指令 (哈哈，这不得是当然的嘛，不然要任务调度做什么，所有任务流水线作业得了)。

因此，下面的描述将借着一次时钟中断，来看一下整一个任务调度操作。

### 定位时钟中断处理逻辑

Intel 8253芯片发起时钟中断之后，CPU 立即开始处理该中断

1. 通过 IDTR 芯片查找中断描述符表(IDT, 最多256项，每项8字节) 的基址。

2. 结合中断号作为偏移量，定位表中某一项具体的中断描述符 (时钟中断是0x20，因此偏移就是 0x20 * 8 = 256)

![](https://ws2.sinaimg.cn/large/006tNbRwgy1fw51zkvo2nj313q0mm3z3.jpg)
<small>Copied from Intel® 64 and IA-32 Architectures Software Developer’s Manual</small>

3. 每一个中断描述符项都将是任务门，中断门，陷阱门三类中的一类，其所占用的 8 字节数据将按如下的形式进行存储

![](https://ws2.sinaimg.cn/large/006tNbRwgy1fw527o8903j314614ita0.jpg)
<small>Copied from Intel® 64 and IA-32 Architectures Software Developer’s Manual</small>

4. 时钟中断是一种中断门，可以看到低 0~15 位和高 48~63 位共同组成了段内偏移量，而低16~31位组成了一个段选择符(可以去 GDT 找到相应的段)。

![](https://ws3.sinaimg.cn/large/006tNbRwgy1fw52a4jbe0j31440zeq3j.jpg)
<small>Copied from Intel® 64 and IA-32 Architectures Software Developer’s Manual</small>

5. 至此，我们就可以定位到处理时钟中断的代码究竟在呢。之后也就是普通的 CPU 继续执行指令的过程(当然，需要注意，这个时候需要注意当前段选择符所表示的 DPL=0 ，即已经进入了内核态)

### 模板式的保存现场 timer\_interrupt

在上一小节第 5 步定位时钟中断的代码时，由于发生了特权级的切换，因此中断处理流程会自动地在新的栈(这里指处理时钟中断使用的内核栈)中保存原来任务的信息，依次入新栈的数据有原任务的 SS, ESP, EFLAGS, CS, EIP, (Error Code) 。

然后将正式进入到 IDT 定位的时钟中断处理逻辑的开始位置 (当然了，到这里也还是在保存现场。毕竟还有好多寄存器的数据要保存的)

```assembly
.align 4
_timer_interrupt:
	push %ds		# save ds,es and put kernel data space
	push %es		# into them. %fs is used by _system_call
	push %fs
	pushl %edx		# we save %eax,%ecx,%edx as gcc doesn't
	pushl %ecx		# save those across function calls. %ebx
	pushl %ebx		# is saved as we use that in ret_sys_call
	pushl %eax
	movl $0x10,%eax # 0x10 是内核数据段选择符，即 GDT 第二项
	mov %ax,%ds
	mov %ax,%es
	movl $0x17,%eax # 0x17 是当前任务数据段选择符，LDT 第二项 (区分选择符是使用 GDT or LDT ，看 val & 0x4 的结果，如果为 1 使用 LDT，否则 GDT
	mov %ax,%fs
	incl _jiffies   # 反正每次时钟中断 +1 ，想不到合适的中文翻译
	movb $0x20,%al		# EOI to interrupt controller #1 发送指令请求硬件结束这次时钟中断的报告
	outb %al,$0x20
	movl CS(%esp),%eax  # 这个 CS 是个常量，取出内核栈暂存的原来正在执行的任务的代码段选择符
	andl $3,%eax		# %eax is CPL (0 or 3, 0=supervisor)    判断一下在中断开始前代码的特权级 0 是内核态，3 是用户态
	pushl %eax
	call _do_timer		# 调用 do_timer(long CPL) 。真正的处理时钟中断
	addl $4,%esp		# task switching to accounting ...
	jmp ret_from_sys_call
```

### 任务调度 do\_timer

```c
void do_timer(long cpl)     /* 这个 cpl 是上一小节获取的原任务正在执行的指令的特权级 */
{
    extern int beepcount;       /* 扬声器发声计数 */
    extern void sysbeepstop(void);  /* 关闭扬声器的函数声明 */

    if (beepcount)              /* 这段逻辑作用很不清晰，可能是富有年代感的产物 */
        if (!--beepcount)
            sysbeepstop();

    if (cpl)                    /* 判断特权级 然后给当前任务的用户态/内核态用时计数 */
        current->utime++;
    else
        current->stime++;

    /* 这段逻辑是给操作系统定时任务用的，用兴趣的欢迎自己学习 */
    if (next_timer) {
        next_timer->jiffies--;
        while (next_timer && next_timer->jiffies <= 0) {
            void (*fn)(void);

            fn = next_timer->fn;
            next_timer->fn = ((void *) 0);
            next_timer = next_timer->next;
            (fn)();
        }
    }
    if (current_DOR & 0xf0)
        do_floppy_timer();
    if ((--current->counter)>0) return;     /* 分配给当前任务的时间片 -1 。如果不为0，那么直接退出时钟中断，让任务继续执行 */
    current->counter=0;     /* 否则，当前任务的时间片记为 0 */
    if (!cpl) return;       /* 如果当前任务正处于内核态(比如用户程序中使用了一些系统调用)，那么先让这个任务继续执行，避免任务切换引起的麻烦 */
    schedule();             /* 任务调用，决定下一个时间片的主人 */
}
```

### 任务调度 schedule()

schedule() 就开始对 CPU 之后要把时间片分配给哪个任务做一次决策了。

但是，在开始之前，我们有必要先了解一下结构体 task\_struct 

```c
struct task_struct {
	long state;	/* -1 unrunnable, 0 runnable, >0 stopped */
	long counter;
	long priority;
	long signal;
	struct sigaction sigaction[32];
	long blocked;	/* bitmap of masked signals */
/* various fields */
	int exit_code;
	unsigned long start_code,end_code,end_data,brk,start_stack;
	long pid,father,pgrp,session,leader;
	unsigned short uid,euid,suid;
	unsigned short gid,egid,sgid;
	long alarm;
	long utime,stime,cutime,cstime,start_time;
	unsigned short used_math;
/* file system info */
	int tty;		/* -1 if no tty, so it must be signed */
	unsigned short umask;
	struct m_inode * pwd;
	struct m_inode * root;
	struct m_inode * executable;
	unsigned long close_on_exec;
	struct file * filp[NR_OPEN];
/* ldt for this task 0 - zero 1 - cs 2 - ds&ss */
	struct desc_struct ldt[3];
/* tss for this task */
	struct tss_struct tss;
};
```

匆匆一瞥，不过是不是觉得有些变量名还是很熟悉的，比如说 priority, utime, uid, eid, gid 等等。
但是暂时还用不上这么多。只要有个概念就好。操作系统通过上述这些值共同维护起了一个任务的方方面面的信息。

其中，需要再次注意的是，Linux0.11 版本最多只支持 64 个任务 (这是硬编码决定的上限，因为这个 task\_struct 结构实例只声明了 64 个)

```c
/*
 *  'schedule()' is the scheduler function. This is GOOD CODE! There
 * probably won't be any reason to change this, as it should work well
 * in all circumstances (ie gives IO-bound processes good response etc).
 * The one thing you might take a look at is the signal-handler code here.
 *
 *   NOTE!!  Task 0 is the 'idle' task, which gets called when no other
 * tasks can run. It can not be killed, and it cannot sleep. The 'state'
 * information in task[0] is never used.
 */
void schedule(void)
{
	int i,next,c;
	struct task_struct ** p;

/* check alarm, wake up any interruptible tasks that have got a signal */
/* 检测 alarm (任务定时报警信号)，并唤醒可中断的任务来完成预定义的工作, 好像自己用得少，还不太了解 alarm(xxx) 的细节
 * 从任务 63 号开始(总共 64 个任务, 0~63) ，倒着遍历所有任务，检测 alarm 
 */
	for(p = &LAST_TASK ; p > &FIRST_TASK ; --p)
		if (*p) {
			if ((*p)->alarm && (*p)->alarm < jiffies) {
					(*p)->signal |= (1<<(SIGALRM-1));
					(*p)->alarm = 0;
				}
			if (((*p)->signal & ~(_BLOCKABLE & (*p)->blocked)) &&
			(*p)->state==TASK_INTERRUPTIBLE)
				(*p)->state=TASK_RUNNING;
		}

/* this is the scheduler proper: */

	while (1) {
		c = -1;
		next = 0;
		i = NR_TASKS;   /* 这里的宏 NR_TASKS = 64 */
		p = &task[NR_TASKS];
		while (--i) {
			if (!*--p)
				continue;
            /* 选择任务状态为 就绪态，且 counter 值最大的任务号作为下一个占用 CPU 的任务
			if ((*p)->state == TASK_RUNNING && (*p)->counter > c) 
				c = (*p)->counter, next = i;
		}
        /* 如果没有其它任务，那么 next = 0 ，即接下来占用 cpu 的任务就是 0号任务(0号任务一定存在，不能被 kill ) */
		if (c) break;
        /* 如果 1~63 号任务(不存在或不在就绪态)至少存在一个任务，且时间片都为 0 ，那么重新计算分配给每个任务的 counter 值 */
		for(p = &LAST_TASK ; p > &FIRST_TASK ; --p)
			if (*p)
				(*p)->counter = ((*p)->counter >> 1) + (*p)->priority;
	}
    /* 将当前任务切换为 next ，然后就可以一切就绪，退出时钟中断，就变成新的占用cpu的任务来执行了 */
	switch_to(next);
}
```

**switch\_to(next)**

switch\_to(next) 这个函数，是通过内嵌汇编写的。下面我们来看看细节(这里选用 GCC 编译出来的汇编指令，不直接使用内嵌汇编)

```assembly
# 这里的汇编指令紧跟在 if(c) break; 之后
.L36:
	movl	%ebx, %edx          # 这里 ebx 存储的是 next 变量的值，现在复制到 edx 上
	sall	$4, %edx            # sall、addl 两行的目的是计算得到一个 TSS 选择符，TSS0 在 GDT 是第4项，TSS1 在 GDT 是第 6 项，类推...其中选择符后3位为DPL和 GDTR/LDTR 选项
	addl	$32, %edx
	movl	_task(,%ebx,4), %ecx    # 找到 next 号任务的 task_struct 的指针 (存储在 task[64] 的数组中)
#APP
# 141 "sched.c" 1
	cmpl %ecx,_current          # 确认是不是原任务
	je 1f                       # 是的话直接跳出 (跳到最近的标签 1 )
	movw %dx,8(%esp)
	xchgl %ecx,_current         # 交换 ecx 和 _current 存储的值
	ljmp 4(%esp)                # 通过间接跳转完成任务切换。通用的形式是 jmp CS:IP，但是使用间接跳转，在内存中的值先读取 32 位偏移量，再读 16 位段选择符
	cmpl %ecx,_last_task_used_math
	jne 1f
	clts                        # 清除 CR0 寄存器中的 TS Flag 
1:
# 0 "" 2
#NO_AP
```

上面这段程序有必要将它割裂成两部分来看待，以 `ljmp 4(%esp)` 为界。

`jmp` ，这是一个相当明显的程序跳转。这里 jmp 给出的选择符是 GDT 上某个任务的 TSS 描述符

通过 `jmp` CPU 将完成老的任务所有寄存器的保存现场，以及新的任务所有寄存器暂存结果的载入。

CPU 执行的下一条指令，将是新的任务 CS:EIP 所指明的指令。

相应的，`ljmp` 之后的指令将在该任务重新获得占用 CPU 后获得执行。

## 任务调度的结束阶段

这段就比较绕了，现在假设一个前提，就是任务 A 也是在时钟中断重新进行任务调度的时候被替换掉的，现在 CPU 分配的时间片又轮到了任务 A 。

也就是紧随着上一小节 `ljmp` 看看从任务内核态回到用户态的流程。

```asm
    # 这段与上面一段指令是直接承接的
	movl	12(%esp), %eax
	xorl	%gs:20, %eax
	je	.L40
	call	___stack_chk_fail
.L40:
	addl	$24, %esp           # 这里返还这次 call 额外申请的内核栈空间
	.cfi_def_cfa_offset 8
	popl	%ebx                # 恢复进入 call 的时候暂存的 ebx
	.cfi_restore 3
	.cfi_def_cfa_offset 4
	ret                         # return 跳转回到当初调用的地方
	.cfi_endproc
```

还是从汇编指令来看

```asm
	call	_schedule           # 原先调用 schedule() 的地方
.L93:
    # return 跳转回来，直接承接的指令
	addl	$8, %esp
	.cfi_def_cfa_offset 8
	popl	%ebx
	.cfi_restore 3
	.cfi_def_cfa_offset 4
	ret                         # 继续 return ，跳出 do_timer() 
	.cfi_endproc
```

```asm
    call _do_timer		# 'do_timer(long CPL)' does everything from
	addl $4,%esp		# task switching to accounting ...
	jmp ret_from_sys_call
```

**ret\_from\_sys\_call**

这里就打算正式结束由于这次时钟中断所引发的内核态指令执行的流程，正式回归用户态了。

```asm
ret_from_sys_call:
    # 这 3 行来判断是不是任务0，如果是就不进行信号量的处理了
	movl _current,%eax		# task[0] cannot have signals
	cmpl _task,%eax
	je 3f
    # 判断在发生时钟中断前，CS 表示的是不是 LDT 第 1 项 (局部变量表的代码段)
    # 否则 CS 就应该内核态代码段了，不进行信号量处理
	cmpw $0x0f,CS(%esp)		# was old code segment supervisor ?
	jne 3f
    # 判断在发生时钟中断前，SS 表示的是不是 LDT 第 2 项 (局部变量表的数据段)
    # 否则认为程序还处在内核态，不进行信号量处理
	cmpw $0x17,OLDSS(%esp)		# was stack segment = 0x17 ?
	jne 3f
    # 设置信号量 (不清楚，先挖个坑)
	movl signal(%eax),%ebx
	movl blocked(%eax),%ecx
	notl %ecx
	andl %ebx,%ecx
	bsfl %ecx,%ecx
	je 3f
	btrl %ecx,%ebx
	movl %ebx,signal(%eax)
	incl %ecx
	pushl %ecx
	call _do_signal
	popl %eax
3:	popl %eax
	popl %ebx
	popl %ecx
	popl %edx
	pop %fs
	pop %es
	pop %ds
	iret
```

至此，通过 iret 将跳转回到用户态代码被中断的位置，并继续执行

## 补充

### pause() 

[Linux Kernel(3) - 操作系统启动](https://dormouse-none.github.io/2018-10-06-understand-Kernel-3/) 中，描述过任务0在完成了整个操作系统的启动流程之后，唯一在做的事情，就是调用 `for(;;) pause();` 

CPU 每次把时间片分配给它，它就开始浪费权力，直接不要了这个时间片。

之前一直以为 pause() 会选择执行 `HLT` 指令让 CPU 暂时地陷入停止状态。但是出乎意料，并不是这个结果(当然，最终是不是还是另一个说法。毕竟代码都是人写的，每个人都可以各自有一套实现)。

当 CPU 把执行指令的权力交个任务 0 时，选择让 CPU 停止，直到接收到硬件中断为止也不失为一种选择。当然，不论其它，Linux0.11版本的代码不是这样的。

通过 `INT 0x80` 配合系统调用函数查表，最后定位到的就是 sys\_pause() 了
```
_sys_pause:
.LFB13:
	.cfi_startproc
	subl	$12, %esp
	.cfi_def_cfa_offset 16
	movl	_current, %eax
	movl	$1, (%eax)
	call	_schedule       # 调用 schedule() 重新划分时间片
	movl	$0, %eax
	addl	$12, %esp
	.cfi_def_cfa_offset 4
	ret
	.cfi_endproc
```

### timer 定时器

之前在 `do_timer()` 函数中也看过了每次时钟中断，都会检查一遍定时器，并决定是否触发预置的定时器处理函数。

当然了，既然是触发定时器，总是要有一个前提——设置定时器

```c
void add_timer(long jiffies, void (*fn)(void))
{
	struct timer_list * p;

	if (!fn)
		return;
	cli();
	if (jiffies <= 0)
		(fn)();
	else {
		for (p = timer_list ; p < timer_list + TIME_REQUESTS ; p++)
			if (!p->fn)
				break;
		if (p >= timer_list + TIME_REQUESTS)
			panic("No more time requests free");
		p->fn = fn;
		p->jiffies = jiffies;
		p->next = next_timer;
		next_timer = p;
		while (p->next && p->next->jiffies < p->jiffies) {
			p->jiffies -= p->next->jiffies;
			fn = p->fn;
			p->fn = p->next->fn;
			p->next->fn = fn;
			jiffies = p->jiffies;
			p->jiffies = p->next->jiffies;
			p->next->jiffies = jiffies;
			p = p->next;
		}
	}
	sti();
}
```

## 小结

对于任务调度，更难的是对一个时间断层的理解。在调度过程中，旧任务被暂停，新的任务被重新启动, 直到旧的任务又被启动。这里就必须人为地将被中断的任务主动的连接起来看待。

早期版本的任务调度模块确实比较简单，累计不过百行 C 代码。哈哈。一点一点继续来吧。

```plain
  __                    __                  
 / _| __ _ _ __   __ _ / _| ___ _ __   __ _ 
| |_ / _` | '_ \ / _` | |_ / _ \ '_ \ / _` |
|  _| (_| | | | | (_| |  _|  __/ | | | (_| |
|_|  \__,_|_| |_|\__, |_|  \___|_| |_|\__, |
                 |___/                |___/ 
```

---
title: 理解 Linux Kernel(10) - Context of Execution
author: fangfeng
date: 2019-04-10
categories:
  - 技术
tags:
  - Linux
  - Kernel
  - COE
---

在进行[第四篇(任务调度)](https://www.ffutop.com/2018-10-12-understand-Kernel-4/)行文描述时，就一直闹不清内核所谓的`task`的概念。之前一直将其与进程(process)的概念等同视之。但这又导致了线程的概念无处安置（毕竟在计算机科学的概念中，线程作为进程的子集存在，负责程序执行）。不过，现在这个疑惑总算得到了合理的解释：**我们错误地将理论和实践不加区分地混淆了**。内核开发社区与学术界的合作在整个内核开发历史上并没有想象中的频繁，正相反，学术界对内核代码的贡献不到1%[1]。如果想要将进程/线程的思想代入内核，并逐一印证，那么过程将非常痛苦并最终一无所获。所谓进程/线程，在内核中只有一个概念——执行的上下文(Context of Execution)，任何想要对进程/线程概念进行区分的行为都将是作茧自缚[2]。同时，`task` 也就是 `Context of Execution` 概念在实现上的表征。

<!--more-->

执行的上下文(Context of Execution, COE)，记录了所有程序执行中的状态，包括：CPU状态（寄存器信息等）、MMU状态（页映射）、权限状态（uid、gid等）和一些公共属性（打开的文件、信号处理函数等）。对比理论上的进程/线程概念，粗略地讲就是同一线程组的线程各自持有CPU状态而共享其它状态，而认为两进程不共享任何状态。当然，在内核的概念里，进程/线程只是COE共享部分状态中的一个特例。

## fork, clone 

如果用进程/线程的概念来看，内核提供了 `fork` 来完成进程的复制，提供了 `clone` 来处理线程的拷贝，另外还有 `vfork` , `kernel_thread` 等。

**syscall fork**
```c
asmlinkage int sys_fork(struct pt_regs regs)
{
    return do_fork(SIGCHLD, regs.esp, &regs, 0, NULL, NULL);
}
```

**syscall clone**
```c
asmlinkage int sys_clone(struct pt_regs regs)
{
    unsigned long clone_flags;
    unsigned long newsp;
    int __user *parent_tidptr, *child_tidptr;
    
    clone_flags = regs.ebx;
    newsp = regs.ecx;
    parent_tidptr = (int __user *)regs.edx;
    child_tidptr = (int __user *)regs.edi;
    if (!newsp)
        newsp = regs.esp;
    return do_fork(clone_flags, newsp, &regs, 0, parent_tidptr, child_tidptr);
}
```

**syscall vfork**
```c
asmlinkage int sys_vfork(struct pt_regs regs)
{
    return do_fork(CLONE_VFORK | CLONE_VM | SIGCHLD, regs.esp, &regs, 0, NULL, NULL);
}
```

**kernel function: kernel\_thread**
```c
int kernel_thread(int (*fn)(void *), void * arg, unsigned long flags)
{
    // ...
    return do_fork(flags | CLONE_VM | CLONE_UNTRACED, 0, &regs, 0, NULL, NULL);
}
```

`fork`, `clone`, `vfork` 的入参怎么和日常使用的系统调用入参不同？且看：

```c
	.macro PTREGSCALL label, func, arg
	.globl \label
\label:
	leaq \func(%rip),%rax
	leaq -ARGOFFSET+8(%rsp),\arg	/* 8 for return address */
	jmp  ia32_ptregs_common
	.endm

	CFI_STARTPROC32

	PTREGSCALL stub32_rt_sigreturn, sys32_rt_sigreturn, %rdi
	PTREGSCALL stub32_sigreturn, sys32_sigreturn, %rdi
	PTREGSCALL stub32_sigaltstack, sys32_sigaltstack, %rdx
	PTREGSCALL stub32_sigsuspend, sys32_sigsuspend, %rcx
	PTREGSCALL stub32_execve, sys32_execve, %rcx
	PTREGSCALL stub32_fork, sys_fork, %rdi
	PTREGSCALL stub32_clone, sys32_clone, %rdx
	PTREGSCALL stub32_vfork, sys_vfork, %rdi
	PTREGSCALL stub32_iopl, sys_iopl, %rsi
	PTREGSCALL stub32_rt_sigsuspend, sys_rt_sigsuspend, %rdx

ENTRY(ia32_ptregs_common)
	popq %r11
	CFI_ENDPROC
	CFI_STARTPROC32	simple
	CFI_SIGNAL_FRAME
	CFI_DEF_CFA	rsp,SS+8-ARGOFFSET
	CFI_REL_OFFSET	rax,RAX-ARGOFFSET
	CFI_REL_OFFSET	rcx,RCX-ARGOFFSET
	CFI_REL_OFFSET	rdx,RDX-ARGOFFSET
	CFI_REL_OFFSET	rsi,RSI-ARGOFFSET
	CFI_REL_OFFSET	rdi,RDI-ARGOFFSET
	CFI_REL_OFFSET	rip,RIP-ARGOFFSET
/*	CFI_REL_OFFSET	cs,CS-ARGOFFSET*/
/*	CFI_REL_OFFSET	rflags,EFLAGS-ARGOFFSET*/
	CFI_REL_OFFSET	rsp,RSP-ARGOFFSET
/*	CFI_REL_OFFSET	ss,SS-ARGOFFSET*/
	SAVE_REST
	call *%rax
	RESTORE_REST
	jmp  ia32_sysret	/* misbalances the return cache */
	CFI_ENDPROC
END(ia32_ptregs_common)
```

进行系统调用时的处理逻辑统一地将这些内容进行了整合，并将CPU状态（所有通用寄存器状态）维护到 `struct pt_regs` 数据块中。总结起来，三种系统调用最终都委托给 `do_fork` 进行实现，只是在参数的选择方面做了不同的预处理。这在用户层的函数原型上就更为明显，`fork` 和 `vfork` 都不允许参数的调用。

```c
pid_t fork(void);

/* for x86-32 */
long clone(unsigned long flags, void *child_stack,
        int *ptid, unsigned long newtls,
        int *ctid);

pid_t vfork(void);
```

再看看 `flags` 有哪些值可选。

```c
/*
 * cloning flags:
 */
#define CSIGNAL		0x000000ff	/* signal mask to be sent at exit */
#define CLONE_VM	0x00000100	/* set if VM shared between processes */
#define CLONE_FS	0x00000200	/* set if fs info shared between processes */
#define CLONE_FILES	0x00000400	/* set if open files shared between processes */
#define CLONE_SIGHAND	0x00000800	/* set if signal handlers and blocked signals shared */
#define CLONE_PTRACE	0x00002000	/* set if we want to let tracing continue on the child too */
#define CLONE_VFORK	0x00004000	/* set if the parent wants the child to wake it up on mm_release */
#define CLONE_PARENT	0x00008000	/* set if we want to have the same parent as the cloner */
#define CLONE_THREAD	0x00010000	/* Same thread group? */
#define CLONE_NEWNS	0x00020000	/* New namespace group? */
#define CLONE_SYSVSEM	0x00040000	/* share system V SEM_UNDO semantics */
#define CLONE_SETTLS	0x00080000	/* create a new TLS for the child */
#define CLONE_PARENT_SETTID	0x00100000	/* set the TID in the parent */
#define CLONE_CHILD_CLEARTID	0x00200000	/* clear the TID in the child */
#define CLONE_DETACHED		0x00400000	/* Unused, ignored */
#define CLONE_UNTRACED		0x00800000	/* set if the tracing process can't force CLONE_PTRACE on this clone */
#define CLONE_CHILD_SETTID	0x01000000	/* set the TID in the child */
#define CLONE_STOPPED		0x02000000	/* Start in stopped state */
#define CLONE_NEWUTS		0x04000000	/* New utsname group? */
#define CLONE_NEWIPC		0x08000000	/* New ipcs */
#define CLONE_NEWUSER		0x10000000	/* New user namespace */
#define CLONE_NEWPID		0x20000000	/* New pid namespace */
#define CLONE_NEWNET		0x40000000	/* New network namespace */
```

`xxx shared between processes` ，描述即将被创建的新任务将进行哪些状态/资源的共享，而哪些状态需要进行拷贝。

### `do_fork`

先看看核心的 `do_fork` 的逻辑。

*Hint: 下列代码经过大量的删减*
```c
long do_fork(unsigned long clone_flags,
        unsigned long stack_start,
        struct pt_regs *regs,
        unsigned long stack_size,
        int __user *parent_tidptr,
        int __user *child_tidptr)
{
    p = copy_process(clone_flags, stack_start, regs, stack_size, child_tidptr, NULL);
    if (!IS_ERR(p)) {
        nr = (clone_flags & CLONE_NEWPID) ?
            task_pid_nr_ns(p, current->nsproxy->pid_ns) :
                task_pid_vnr(p);
        if (!(clone_flags & CLONE_STOPPED))
            wake_up_new_task(p, clone_flags);
        else
            p->state = TASK_STOPPED;
    } else {
        nr = PTR_ERR(p);
    }
    return nr;
}
```

```c
static struct task_struct *copy_process(unsigned long clone_flags,
                    unsigned long stack_start,
                    struct pt_regs *regs,
                    unsigned long stack_size,
                    int __user *child_tidptr,
                    struct pid *pid)
{
    /* 预分配 task_struct 数据结构空间 */
    retval = security_task_create(clone_flags);
    /* 复制 current 的 task_struct 数据结构 */
    p = dup_task_struct(current);

    if (nr_threads >= max_threads)
    	goto bad_fork_cleanup_count;

    /* 针对多核CPU，为新任务分配CPU */
    sched_fork(p, clone_flags);
    /* 复制 thread_info 数据结构及线程栈 */
    retval = copy_thread(0, clone_flags, stack_start, stack_size, p, regs);

    /* 分配新的 pid */
    p->pid = pid_nr(pid);
    /* thread group id = new pid */
    p->tgid = p->pid;
    /* 如果标志是 CLONE_THREAD，tgid = 父任务id */
    if (clone_flags & CLONE_THREAD)
    	p->tgid = current->tgid;

    p->set_child_tid = (clone_flags & CLONE_CHILD_SETTID) ? child_tidptr : NULL;
    p->clear_child_tid = (clone_flags & CLONE_CHILD_CLEARTID) ? child_tidptr: NULL;
    /*
     * sigaltstack should be cleared when sharing the same VM
     */
    if ((clone_flags & (CLONE_VM|CLONE_VFORK)) == CLONE_VM)
    	p->sas_ss_sp = p->sas_ss_size = 0;

    /* ok, now we should be set up.. */
    p->exit_signal = (clone_flags & CLONE_THREAD) ? -1 : (clone_flags & CSIGNAL);
    p->pdeath_signal = 0;
    p->exit_state = 0;

    /*
     * Ok, make it visible to the rest of the system.
     * We dont wake it up yet.
     */
    p->group_leader = p;
    INIT_LIST_HEAD(&p->thread_group);
    INIT_LIST_HEAD(&p->ptrace_children);
    INIT_LIST_HEAD(&p->ptrace_list);

    p->cpus_allowed = current->cpus_allowed;
    if (unlikely(!cpu_isset(task_cpu(p), p->cpus_allowed) ||
    		!cpu_online(task_cpu(p))))
    	set_task_cpu(p, smp_processor_id());

    /* CLONE_PARENT re-uses the old parent */
    if (clone_flags & (CLONE_PARENT|CLONE_THREAD))
    	p->real_parent = current->real_parent;
    else
    	p->real_parent = current;
    p->parent = p->real_parent;

    spin_lock(&current->sighand->siglock);

    if (clone_flags & CLONE_THREAD) {
    	p->group_leader = current->group_leader;
    	list_add_tail_rcu(&p->thread_group, &p->group_leader->thread_group);

    	if (!cputime_eq(current->signal->it_virt_expires,
    			cputime_zero) ||
    	    !cputime_eq(current->signal->it_prof_expires,
    			cputime_zero) ||
    	    current->signal->rlim[RLIMIT_CPU].rlim_cur != RLIM_INFINITY ||
    	    !list_empty(&current->signal->cpu_timers[0]) ||
    	    !list_empty(&current->signal->cpu_timers[1]) ||
    	    !list_empty(&current->signal->cpu_timers[2])) {
    		/*
    		 * Have child wake up on its first tick to check
    		 * for process CPU timers.
    		 */
    		p->it_prof_expires = jiffies_to_cputime(1);
    	}
    }

    if (likely(p->pid)) {
    	add_parent(p);
    	if (unlikely(p->ptrace & PT_PTRACED))
    		__ptrace_link(p, current->parent);

    	if (thread_group_leader(p)) {
    		if (clone_flags & CLONE_NEWPID)
    			p->nsproxy->pid_ns->child_reaper = p;

    		p->signal->tty = current->signal->tty;
    		set_task_pgrp(p, task_pgrp_nr(current));
    		set_task_session(p, task_session_nr(current));
    		attach_pid(p, PIDTYPE_PGID, task_pgrp(current));
    		attach_pid(p, PIDTYPE_SID, task_session(current));
    		list_add_tail_rcu(&p->tasks, &init_task.tasks);
    		__get_cpu_var(process_counts)++;
    	}
    	attach_pid(p, PIDTYPE_PID, pid);
    	nr_threads++;
    }

    total_forks++;
    spin_unlock(&current->sighand->siglock);
    write_unlock_irq(&tasklist_lock);
    proc_fork_connector(p);
    cgroup_post_fork(p);
    return p;
}
```

根据入参配置的 `flags` ，`copy_process` 确定了新任务与父任务共享的状态，以及一些需要拷贝的状态。

- `fork` 产生一个新的任务，与父任务不存在任何资源共享的情况。
- `clone` 可高度定制化的系统调用，几乎可以自由组合定制新的任务
- `vfork` 历史原因而存在的系统调用，设计目的在于一般 `fork` 之后都将调用 `execve` 来执行全新的任务，也就导致了 `fork` 所做的拷贝全部白费，因此搞了个轻量级的 `vfork` 来避免做内存的拷贝。

> **VFORK** 
> Historic description
> Under Linux, fork(2) is implemented using copy-on-write pages, so the only penalty incurred by fork(2) is the time and memory required to duplicate the  parent's page tables, and to create a unique task structure for the child.  However, in the bad old days a fork(2) would require making a complete copy of the caller's data space, often needlessly, since usually immediately afterward an exec(3) is done.  Thus, for greater efficiency, BSD introduced  the  vfork() system call, which did not fully copy the address space of the parent process, but borrowed the parent's memory and thread of control until a call to execve(2) or an exit occurred.  The parent process was suspended while the child was using its resources.   The  use  of vfork() was tricky: for example, not modifying data in the parent process depended on knowing which variables were held in a register.

## pid, tgid

```c
/* 分配新的 pid */
p->pid = pid_nr(pid);
/* thread group id = new pid */
p->tgid = p->pid;
/* 如果标志是 CLONE_THREAD，tgid = 父任务id */
if (clone_flags & CLONE_THREAD)
	p->tgid = current->tgid;
```

`pid` 作为每个 `task` 的唯一标识符存在。`tgid` 用于指代线程组id，从代码上看也就是进程主线程id。（请原谅我又用了进程/线程的表述）

看着没有问题？当然不可能。这段代码可是意味着 `pid` 唯一啊，这可不符合日常表述啊。对于系统的终端用户来讲，一个进程可以有一个或多个线程。`pid` 可是一直被翻译成进程ID(process id)。难道？

这就是本质实现与表面功夫的差别啦。`pid_t getpid(void);`, `pid_t gettid(void);` 两个系统调用分别被用来提供进程ID和线程ID（请仔细思考下代码实现和对外展示的差距）。

```c
/**
 * sys_getpid - return the thread group id of the current process
 *
 * Note, despite the name, this returns the tgid not the pid.  The tgid and
 * the pid are identical unless CLONE_THREAD was specified on clone() in
 * which case the tgid is the same in all threads of the same group.
 *
 * This is SMP safe as current->tgid does not change.
 */
asmlinkage long sys_getpid(void)
{
	return task_tgid_vnr(current);
}
```

```c
/* Thread ID - the internal kernel "pid" */
asmlinkage long sys_gettid(void)
{
	return task_pid_vnr(current);
}
```

这样就好懂多了吧。用户态通过系统调用取到的 `pid`, `tid` 已经经过了一层加工，分别映射着内核实现的 `tgid`, `pid` 。

*额外地：想通过 `ps` 查看进程/线程可以使用 `ps -eLf`*

## schedule

再来回顾下任务调度是如何实现的。与[第四篇 任务调度](https://www.ffutop.com/2018-10-12-understand-Kernel-4/)描述的并无太大不同。要说最大的区分，就是早期版本与时下版本顺时的演进。当然，还有就是多核CPU下的SMP调度。

至于多核下的所谓进程/线程如何调度。内核根本认识不到进程/线程的概念，唯一的只有任务(task, or COE)。而每个用户所认为的线程在内核的认识下也是一个任务。因此，所谓用户认识到的多线程并发执行在多核CPU下也就完全是可行的。

## concept of Thread

且不论Linux内核没有线程概念的前提，线程的定义总是存在的，而满足其定义的实现，也就可以被称为“线程”。哪些要素组成了线程呢？

1. 线程是与其它代码共享进程地址空间的最小执行流
2. 诸如栈、寄存器信息、本地线程数据需要保持独立
3. 互斥锁(Mutex)、条件变量(Condition Variable)、线程间同步管理的支持
4. ...

既然都说了Linux内核没有线程概念，现在却又开始提，这不是打脸嘛？OK，具体情况是：内核本身不支持线程，甚至没有线程概念。但其通过GLIBC的NPTL(Native POSIX Thread Library)支持了线程。

> 在Linux内核2.6出现之前任务是(最小)可调度的对象，当时的Linux不真正支持线程。但是Linux内核有一个系统调用指令`clone()`，这个指令将产生一个调用方任务的拷贝，而且这个拷贝可以与原任务使用同一地址空间。LinuxThreads计划使用这个系统调用来提供一个内核级别的线程支持。但是这个解决方法与真正的POSIX标准有一些不兼容的地方，尤其是在信号处理、进程调度和进程间同步原语方面。
> *Copied From [Wikipedia NPTL](https://en.wikipedia.org/wiki/Native_POSIX_Thread_Library)*

![Linux 架构](https://ws4.sinaimg.cn/large/006tNc79ly1g1x9mjuahnj31400u0doi.jpg)

## Thread Model

最后，聊聊线程模型。相信各位不管懂或不懂，至少都听说过 1:1, N:1, M:N 线程模型。至于究竟这些模型是怎么回事呢？且看：

![Thread Model](https://ws4.sinaimg.cn/large/006tNc79ly1g1xhulou5bj30go08zglv.jpg)

### 1:1 Model

最简单的线程模型，也是内核通过NPTL最容易实现的一种模型。每个通过 `clone` 产生的新任务（当然，至少要符合共享内存等线程定义）对应用程序来说就是一个新线程。所谓的用户级线程和内核级线程，在 1:1 模型中就是同一个任务，只是新任务在被内核 `clone` 之后，又由NPTL做了进一步的管理，因此在形式上看似有用户级/内核级之分。

### N:1 Model

N:1 线程模型又称为“用户级线程模型”。在这种模型下，线程的概念由用户态代码进行支持，包括用来管理线程的用户空间调度器以及以非阻塞模式捕捉和处理I/O机制。使用N:1模型的好处在于其上下文切换的代价几乎为零，毕竟这是纯用户态的。当然，缺点也比较明显：由于对内核来说，认为这只是单个任务，也就无法充分利用多核CPU的特性。

### M:N Model

有没有方式整合1:1模型和N:1模型，让内核意识到有几个任务，但又存在更多的线程在用户级执行呢？当然也是可行，但如何协调调度又是费事费力的实现了。

## Threads vs Events

线程怎么又和事件扯上关系了呢？先来回顾下任务调度的几种模式：

1. 串行，每个任务依次执行，不存在任务调度
2. 抢占式，通过时钟中断决定是否切换任务
3. 协作式，只有当任务主动放弃对CPU的占用，才能轮到下一个任务。

对于用户级线程来说，内核无法意识到这些线程的存在，硬件发起的所有中断又都被内核捕获。因此，想要实现抢占式是不可能完成的任务（至少以本人目前所知是不可能的）。故协作式线程调度是用户级线程的唯一选择。而协作式就意味着是由线程硬编码主动放弃任务执行（`yield` / `schedule` / etc.）。如此这般，与事件驱动下，不同代码片之间的调度又是何其相似。当然，只是开个玩笑，如果都一样了，直接用事件驱动不就OK了。线程之所以是线程，有其独特的定义在。不过不再细说，可以参考 [Threads vs Events](https://courses.cs.vt.edu/cs5204/fall09-kafura/Presentations/Threads-VS-Events.pdf)

## 参考

\[1\]. Mauerer W. Professional Linux kernel architecture[M]. John Wiley & Sons, 2010.
\[2\]. [Linus Torvalds. Re: proc fs and shared pids](https://lkml.iu.edu/hypermail/linux/kernel/9608/0191.html)[EL/OL]. Aug 6th, 1996. 
\[3\]. [Multithreaded Programming (POSIX pthreads Tutorial)](https://randu.org/tutorials/threads/)[EL/OL].
\[4\]. [线程模型](https://blog.csdn.net/u012432778/article/details/47378321)[EL/OL].
\[5\]. [Implementing a Thread Library on Linux](https://www.evanjones.ca/software/threading.html)[EL/OL]. Dec 10th, 2003.

## 用户级线程资料参考

\[1\]. `man makecontext` & `man swapcontext`
\[2\]. [Libfiber](https://github.com/brianwatling/libfiber)
\[3\]. [**所谓的**抢占式用户线程实现](https://github.com/dramesh/GTThreads)

```plain
  __                    __                  
 / _| __ _ _ __   __ _ / _| ___ _ __   __ _ 
| |_ / _` | '_ \ / _` | |_ / _ \ '_ \ / _` |
|  _| (_| | | | | (_| |  _|  __/ | | | (_| |
|_|  \__,_|_| |_|\__, |_|  \___|_| |_|\__, |
                 |___/                |___/ 
```

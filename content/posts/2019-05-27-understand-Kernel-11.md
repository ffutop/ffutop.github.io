---
title: 理解 Linux Kernel (11) - 进程间通信
author: fangfeng
date: 2019-05-27
categories:
  - 技术
tags:
  - Linux
  - Kernel
  - IPC
---

进程间通信(IPC, inter-process communication)是多个执行上下文实现数据交互的重要功能，也是 Linux Kernel 一个重要的模块。本篇主要着眼于 Linux 基于 System V 引入的 3 种 IPC 机制——信号量、消息队列、共享内存。除此之外，Linux 还有更多的方式能够实现进程间通信，但本文不做介绍。

*本篇基于 Linux 4.9.87 版本源码*

<!--more-->

## IPC 命名空间

读过前几篇的同学应该都能够了解，内核统一维护了所有的资源，并有限地向各个执行流暴露所需的资源。这保证了各个执行流看似独立地运行，但也为进程间的协作制造了障碍。基于这个原因，内核又提供了额外的机制来超脱这个限制。

```c
struct kern_ipc_perm
{
	spinlock_t	lock;
	bool		deleted;
	int		id;             /* 内核态内部 id */
	key_t		key;        /* 保存用户程序用来唯一区分信号量的一个魔数 */
	kuid_t		uid;
	kgid_t		gid;
	kuid_t		cuid;
	kgid_t		cgid;
	umode_t		mode;
	unsigned long	seq;
	void		*security;
};

struct ipc_ids {
	int in_use;                 /* 使用中的 IPC 对象数量 */
	unsigned short seq;         /* 用户空间 IPC 对象 ID */
	struct rw_semaphore rwsem;  /* 内核信号量，内核操作必备的锁机制 */
	struct idr ipcs_idr;        /* id 管理器，ipcs_idr 总是指向 kern_ipc_perm 对象 */
	int next_id;
};

struct ipc_namespace {
	atomic_t	count;          /* 被引用的次数 */
    /* “核心”，每个数组元素对应一种 IPC 机制：信号量、消息队列、共享内存 */
    /*
       from ipc/util.h
        #define IPC_SEM_IDS 0   信号量
        #define IPC_MSG_IDS 1   消息队列
        #define IPC_SHM_IDS 2   共享内存
    */
	struct ipc_ids	ids[3];

    /* 以下都是对三种 IPC 机制设置的限制，诸如共享内存页的最大数量等 */
	int		sem_ctls[4];
	int		used_sems;

	unsigned int	msg_ctlmax;
	unsigned int	msg_ctlmnb;
	unsigned int	msg_ctlmni;
	atomic_t	msg_bytes;
	atomic_t	msg_hdrs;

	size_t		shm_ctlmax;
	size_t		shm_ctlall;
	unsigned long	shm_tot;
	int		shm_ctlmni;
	int		shm_rmid_forced;

	struct notifier_block ipcns_nb;

	/* The kern_mount of the mqueuefs sb.  We take a ref on it */
	struct vfsmount	*mq_mnt;

	/* # queues in this ns, protected by mq_lock */
	unsigned int    mq_queues_count;

	/* next fields are set through sysctl */
	unsigned int    mq_queues_max;   /* initialized to DFLT_QUEUESMAX */
	unsigned int    mq_msg_max;      /* initialized to DFLT_MSGMAX */
	unsigned int    mq_msgsize_max;  /* initialized to DFLT_MSGSIZEMAX */
	unsigned int    mq_msg_default;
	unsigned int    mq_msgsize_default;

	/* user_ns which owns the ipc ns */
	struct user_namespace *user_ns;
	struct ucounts *ucounts;

	struct ns_common ns;
};
```

IPC 由一个被称为 `ipc_namespace` 的数据结构维护，如果熟悉 `pid_namespace` 之类的，应该能够理解内核通过命名空间实现了资源的隔离。这里的 `ipc_namespace` 也是为了实现多个 ipc 环境所做的抽象。首先看下初始化 IPC 命名空间的流程。

```c
/* from init/init_task.c */
/* 内核抽象的第一个任务 */
struct task_struct init_task = INIT_TASK(init_task);

/* from include/linux/init_task.h */
#define INIT_TASK(tsk) \
{                                       \
    .nsproxy	= &init_nsproxy,        \
}

/* from kernel/nsproxy.c */
struct nsproxy init_nsproxy = {
	.count			= ATOMIC_INIT(1),
	.uts_ns			= &init_uts_ns,
#if defined(CONFIG_POSIX_MQUEUE) || defined(CONFIG_SYSVIPC)
	.ipc_ns			= &init_ipc_ns,
#endif
	.mnt_ns			= NULL,
	.pid_ns_for_children	= &init_pid_ns,
#ifdef CONFIG_NET
	.net_ns			= &init_net,
#endif
#ifdef CONFIG_CGROUPS
	.cgroup_ns		= &init_cgroup_ns,
#endif
};

/* from ipc/msgutil.c */ 
struct ipc_namespace init_ipc_ns = {
	.count		= ATOMIC_INIT(1),
	.user_ns = &init_user_ns,
	.ns.inum = PROC_IPC_INIT_INO,
#ifdef CONFIG_IPC_NS
	.ns.ops = &ipcns_operations,
#endif
};
```

任务通过 `fork`、`clone` 等操作构建新的任务，并由 `flag CLONE_NEWIPC` 决定与父任务共享 IPC 命名空间或创建新的 IPC 命名空间。

```c
struct ipc_namespace *copy_ipcs(unsigned long flags,
	struct user_namespace *user_ns, struct ipc_namespace *ns)
{
	if (!(flags & CLONE_NEWIPC))
        /* 返回原来的 IPC 命名空间 */
		return get_ipc_ns(ns);
    /* 返回一个新的 IPC 命名空间 */
	return create_ipc_ns(user_ns, ns);
}
```

## 信号量

![](https://img.ffutop.com/7A24F185-3480-4C29-A006-C66BB54E046F.png)

上图给出了各个结构间的关系，通过当前任务指向的 IPC 命名空间，找到 `struct ipc_ids` ，内核通过 `ipcs_idr` 找到 ID 到指针的映射关系，从而得到所需的 `kern_ipc_perm` 实例。同时 `kern_ipc_perm` 作为结构 `sem_array` 的第一个元素，使用技巧就可以直接定位到 `struct sem_array` 。

### syscall `semget`

获取一个信号量

```c
/* from ipc/sem.c */
SYSCALL_DEFINE3(semget, key_t, key, int, nsems, int, semflg)
{
	struct ipc_namespace *ns;
	static const struct ipc_ops sem_ops = {
		.getnew = newary,
		.associate = sem_security,
		.more_checks = sem_more_checks,
	};
	struct ipc_params sem_params;

    /* 获取当前任务的 IPC 命名空间 */
	ns = current->nsproxy->ipc_ns;

	if (nsems < 0 || nsems > ns->sc_semmsl)
		return -EINVAL;

	sem_params.key = key;
	sem_params.flg = semflg;
	sem_params.u.nsems = nsems;

	return ipcget(ns, &sem_ids(ns), &sem_ops, &sem_params);
}

/* from ipc/util.c */
int ipcget(struct ipc_namespace *ns, struct ipc_ids *ids,
			const struct ipc_ops *ops, struct ipc_params *params)
{
    /* 如果标志位为“私有”，则创建新的 IPC 命名空间 */
	if (params->key == IPC_PRIVATE)
		return ipcget_new(ns, ids, ops, params);
    /* 使用当前任务的 IPC 命名空间 */
	else
		return ipcget_public(ns, ids, ops, params);
}
```

![](https://img.ffutop.com/DED2216C-127B-4036-84D8-33A2E44C5B33.png)

这里额外展示了 `msgget`, `shmget`，分别意味着消息队列、共享内存。获取 IPC 对象的流程都是相同的，有所区别的只是获取、写入和存储数据的方式。

```c
// from 'ipc/util.c'
int ipcget_new(struct ipc_namespace *ns, struct ipc_ids *ids,
		struct ipc_ops *ops, struct ipc_params *params)
{
	int err;
retry:
	err = idr_pre_get(&ids->ipcs_idr, GFP_KERNEL);

	if (!err)
		return -ENOMEM;

	down_write(&ids->rw_mutex);
	err = ops->getnew(ns, params);
	up_write(&ids->rw_mutex);

	if (err == -EAGAIN)
		goto retry;

	return err;
}

int ipcget_public(struct ipc_namespace *ns, struct ipc_ids *ids,
		struct ipc_ops *ops, struct ipc_params *params)
{
	struct kern_ipc_perm *ipcp;
	int flg = params->flg;
	int err;
retry:
	err = idr_pre_get(&ids->ipcs_idr, GFP_KERNEL);

  // 读写锁保护临界区
	down_write(&ids->rw_mutex);
  // 确认 KEY 是否存在
	ipcp = ipc_findkey(ids, params->key);
	if (ipcp == NULL) {
		/* KEY 不存在，创建新的 */
		if (!(flg & IPC_CREAT))
			err = -ENOENT;
		else if (!err)
			err = -ENOMEM;
		else
			err = ops->getnew(ns, params);
	} else {
		/* KEY 存在，check ACL */
		if (flg & IPC_CREAT && flg & IPC_EXCL)
			err = -EEXIST;
		else {
			err = 0;
			if (ops->more_checks)
				err = ops->more_checks(ipcp, params);
			if (!err)
				err = ipc_check_perms(ipcp, ops, params);
		}
		ipc_unlock(ipcp);
	}
  // 离开临界区
	up_write(&ids->rw_mutex);

	if (err == -EAGAIN)
		goto retry;

	return err;
}
```

`ipcget_new` 和 `ipcget_public` ，核心的目标用 key 去换取一个 id (代表 `struct kern_ipc_perm` 在 `ipcs_idr` 整数指针管理器中的 id )。

如果这个 key 不存在，就考虑创建一个新的 `struct kern_ipc_perm` 。

```c
// from 'include/linux/sem.h'
struct sem_array {
	struct kern_ipc_perm	sem_perm;	/* permissions .. see ipc.h */
	time_t			sem_otime;	/* last semop time */
	time_t			sem_ctime;	/* last change time */
	struct sem		*sem_base;	/* ptr to first semaphore in array */
	struct sem_queue	*sem_pending;	/* pending operations to be processed */
	struct sem_queue	**sem_pending_last; /* last pending operation */
	struct sem_undo		*undo;		/* undo requests on this array */
	unsigned long		sem_nsems;	/* no. of semaphores in array */
};

// from 'ipc/sem.c'
static int newary(struct ipc_namespace *ns, struct ipc_params *params)
{
	int id;
	int retval;
	struct sem_array *sma;
	int size;
	key_t key = params->key;
	int nsems = params->u.nsems;
	int semflg = params->flg;

	if (!nsems)	// 信号量集合中信号量数量不能为0.
		return -EINVAL;
	if (ns->used_sems + nsems > ns->sc_semmns) // 不能超过 IPC 信号量数量上限
		return -ENOSPC;

	size = sizeof (*sma) + nsems * sizeof (struct sem);
	sma = ipc_rcu_alloc(size);
	if (!sma) {
		return -ENOMEM;
	}
	memset (sma, 0, size);

	sma->sem_perm.mode = (semflg & S_IRWXUGO);
	sma->sem_perm.key = key;

	sma->sem_perm.security = NULL;
	retval = security_sem_alloc(sma);
	if (retval) {
		ipc_rcu_putref(sma);
		return retval;
	}

	id = ipc_addid(&sem_ids(ns), &sma->sem_perm, ns->sc_semmni);
	if (id < 0) {
		security_sem_free(sma);
		ipc_rcu_putref(sma);
		return id;
	}
	ns->used_sems += nsems;

	sma->sem_perm.id = sem_buildid(id, sma->sem_perm.seq);
	sma->sem_base = (struct sem *) &sma[1];
	/* sma->sem_pending = NULL; */
	sma->sem_pending_last = &sma->sem_pending;
	/* sma->undo = NULL; */
	sma->sem_nsems = nsems;
	sma->sem_ctime = get_seconds();
	sem_unlock(sma);

	return sma->sem_perm.id;
}
```

### syscall `semctl`

信号量控制操作。根据 `cmd` 的不同，可以获得信号集合中所有值(GETALL)，获取最后一个操作信号集合的进程号(GETPID)，统一设置信号集合中所有值(SETALL)

```c
SYSCALL_DEFINE4(semctl, int, semid, int, semnum, int, cmd, unsigned long, arg)
{
	int version;
	struct ipc_namespace *ns;
	void __user *p = (void __user *)arg;

	if (semid < 0)
		return -EINVAL;

	version = ipc_parse_version(&cmd);
	ns = current->nsproxy->ipc_ns;

	switch (cmd) {
	case IPC_INFO:
	case SEM_INFO:
	case IPC_STAT:
	case SEM_STAT:
		return semctl_nolock(ns, semid, cmd, version, p);
	case GETALL:
	case GETVAL:
	case GETPID:
	case GETNCNT:
	case GETZCNT:
	case SETALL:
		return semctl_main(ns, semid, semnum, cmd, p);
	case SETVAL:
		return semctl_setval(ns, semid, semnum, arg);
	case IPC_RMID:
	case IPC_SET:
		return semctl_down(ns, semid, cmd, version, p);
	default:
		return -EINVAL;
	}
}
```

### syscall `semop`

信号的 P/V 操作

```c
// from 'ipc/sem.c'
asmlinkage long sys_semop (int semid, struct sembuf __user *tsops, unsigned nsops)
{
    return sys_semtimedop(semid, tsops, nsops, NULL);
}

asmlinkage long sys_semtimedop(int semid, struct sembuf __user *tsops,
			unsigned nsops, const struct timespec __user *timeout)
{
	int error = -EINVAL;
	struct sem_array *sma;
	struct sembuf fast_sops[SEMOPM_FAST];
	struct sembuf* sops = fast_sops, *sop;
	struct sem_undo *un;
	int undos = 0, alter = 0, max;
	struct sem_queue queue;
	unsigned long jiffies_left = 0;
	struct ipc_namespace *ns;

  // 获取当前任务的 ipc 命名空间
	ns = current->nsproxy->ipc_ns;

	if (nsops < 1 || semid < 0)
		return -EINVAL;
	if (nsops > ns->sc_semopm)
		return -E2BIG;
	if(nsops > SEMOPM_FAST) {
		sops = kmalloc(sizeof(*sops)*nsops,GFP_KERNEL);
		if(sops==NULL)
			return -ENOMEM;
	}
	if (copy_from_user (sops, tsops, nsops * sizeof(*tsops))) {
		error=-EFAULT;
		goto out_free;
	}
	if (timeout) {
		struct timespec _timeout;
		if (copy_from_user(&_timeout, timeout, sizeof(*timeout))) {
			error = -EFAULT;
			goto out_free;
		}
		if (_timeout.tv_sec < 0 || _timeout.tv_nsec < 0 ||
			_timeout.tv_nsec >= 1000000000L) {
			error = -EINVAL;
			goto out_free;
		}
		jiffies_left = timespec_to_jiffies(&_timeout);
	}
	max = 0;
	for (sop = sops; sop < sops + nsops; sop++) {
		if (sop->sem_num >= max)
			max = sop->sem_num;
		if (sop->sem_flg & SEM_UNDO)
			undos = 1;
		if (sop->sem_op != 0)
			alter = 1;
	}

retry_undos:
	if (undos) {
		un = find_undo(ns, semid);
		if (IS_ERR(un)) {
			error = PTR_ERR(un);
			goto out_free;
		}
	} else
		un = NULL;

	sma = sem_lock_check(ns, semid);
	if (IS_ERR(sma)) {
		error = PTR_ERR(sma);
		goto out_free;
	}

	/*
	 * semid identifiers are not unique - find_undo may have
	 * allocated an undo structure, it was invalidated by an RMID
	 * and now a new array with received the same id. Check and retry.
	 */
	if (un && un->semid == -1) {
		sem_unlock(sma);
		goto retry_undos;
	}
	error = -EFBIG;
	if (max >= sma->sem_nsems)
		goto out_unlock_free;

	error = -EACCES;
	if (ipcperms(&sma->sem_perm, alter ? S_IWUGO : S_IRUGO))
		goto out_unlock_free;

	error = security_sem_semop(sma, sops, nsops, alter);
	if (error)
		goto out_unlock_free;

	error = try_atomic_semop (sma, sops, nsops, un, task_tgid_vnr(current));
	if (error <= 0) {
		if (alter && error == 0)
			update_queue (sma);
		goto out_unlock_free;
	}

	/* We need to sleep on this operation, so we put the current
	 * task into the pending queue and go to sleep.
	 */

	queue.sma = sma;
	queue.sops = sops;
	queue.nsops = nsops;
	queue.undo = un;
	queue.pid = task_tgid_vnr(current);
	queue.id = semid;
	queue.alter = alter;
	if (alter)
		append_to_queue(sma ,&queue);
	else
		prepend_to_queue(sma ,&queue);

	queue.status = -EINTR;
	queue.sleeper = current;
	current->state = TASK_INTERRUPTIBLE;
	sem_unlock(sma);

	if (timeout)
		jiffies_left = schedule_timeout(jiffies_left);
	else
		schedule();

	error = queue.status;
	while(unlikely(error == IN_WAKEUP)) {
		cpu_relax();
		error = queue.status;
	}

	if (error != -EINTR) {
		/* fast path: update_queue already obtained all requested
		 * resources */
		goto out_free;
	}

	sma = sem_lock(ns, semid);
	if (IS_ERR(sma)) {
		BUG_ON(queue.prev != NULL);
		error = -EIDRM;
		goto out_free;
	}

	/*
	 * If queue.status != -EINTR we are woken up by another process
	 */
	error = queue.status;
	if (error != -EINTR) {
		goto out_unlock_free;
	}

	/*
	 * If an interrupt occurred we have to clean up the queue
	 */
	if (timeout && jiffies_left == 0)
		error = -EAGAIN;
	remove_from_queue(sma,&queue);
	goto out_unlock_free;

out_unlock_free:
	sem_unlock(sma);
out_free:
	if(sops != fast_sops)
		kfree(sops);
	return error;
}
```

## 消息队列

至于消息队列和共享内存，都利用了类似的技术，通过在 IPC 命名空间下，使用 `struct idr` 做 KEY, VALUE 的管理，从而维护起了一系列互不干扰的消息队列、共享内存。

![](https://img.ffutop.com/409ECF12-A0E5-4B0F-AD96-FE9EA73FD972.png)

## 共享内存

![](https://img.ffutop.com/85BB67B8-0EE1-4657-8242-760645A06ED1.png)

## 结束

局限于目前未能找到 SysV IPC 的经典利用场景，加之平时工作接触甚少。本篇匆匆结尾，未详细展示不同 IPC 技术在不同使用场景下的优劣。

相对而言，日常更多使用的 IPC 技术反而是管道、多进程共同读写文档等。

此坑估计不会再填了...

```plain
  __                    __                  
 / _| __ _ _ __   __ _ / _| ___ _ __   __ _ 
| |_ / _` | '_ \ / _` | |_ / _ \ '_ \ / _` |
|  _| (_| | | | | (_| |  _|  __/ | | | (_| |
|_|  \__,_|_| |_|\__, |_|  \___|_| |_|\__, |
                 |___/                |___/ 
```

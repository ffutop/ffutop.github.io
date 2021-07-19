---
title: Linux Traffic Control
author: fangfeng
date: 2019-08-23
categories:
  - 技术
tags:
  - Linux
  - Kernel
  - TC
---

最近在处理 Kubernetes 工作的时候，被问及这样一个命题：Pod 能对 CPU 和内存施加限制，那同样属于资源范畴的网络带宽是否能限制呢？使用 Kubernetes 的一个核心优势在于每个 Pod 都等同于一个轻量级的“操作系统”。建立在 Linux 命名空间(namespace)和控制组(cgroups)基础上的容器技术将每个 Pod 的资源进行了隔离和限制。但是，限流只针对 CPU 和内存，对网络、磁盘 IO 的解决方案仅仅局限在隔离，难道技术上实现不了吗？自然不是，Kubernetes 有意识地将网络模块拆解，只定义插件规范，而将实现的可能性交由下游开发自由决策。当然，本篇并不在意 Kubernetes 网络限流的解决方案，只是以此作为引子。

流量控制(Traffic Control, TC) 是 Linux 内核提供的流量限速、整形、策略控制机制。近乎完美地支持网络限流的命题，除了，这是比 Netfilter 更难理解的模块。Netfilter 作用在内核网络协议栈上，通过在各个枢纽设立关卡对网络包(`sk_buff` 数据结构, skb)进行检查，并实施 ACCEPT、DROP、MASQUERADE 等策略。相比之下，TC 是绑定在网络设备上实施的。提供 `enqueue`, `dequeue` 两个核心函数，也是作为关卡对到达网络设备的网络包实施策略。要说核心的不同之处，Netfilter 是流式地处理网络包，先到的网络包一定先出（也可能是被丢弃）；TC 的处理方式就依照策略，类比块设备的随机读/随机写了。

<!--more-->

## 前置知识：发送网络包流程

此处提供一个内核网络包的发送流程，辅助了解网络协议栈。*跳过不影响后文阅读*

网络包的发送由应用层通过系统调用 `sendmsg` (类似的系统调用还有 `send`, `sendto` 等) 发起，后陷入内核态，由内核代码实现网络协议栈逐层封装，逐层下发的工作。

```plain
 0)               |              sock_sendmsg() {       /* 系统调用 sendmsg 对应的内核函数 */
 0)               |                inet_sendmsg() {     /* 使用 IPv4 (not IPv6) 实现的 sendmsg */
 0)               |                  tcp_sendmsg() {    /* 传输层，TCP 协议的处理 */
 0)               |                      sk_stream_alloc_skb() {    /* 创建 sk_buff(skb) 数据结构，网络包在内核中存在的形式 */
 0)               |                        __alloc_skb() {
 0)   5.273 us    |                        }
 0)   0.056 us    |                        sk_forced_mem_schedule();
 0)   6.026 us    |                      }
 0)               |                      tcp_push() {
 0)               |                        __tcp_push_pending_frames() {
 0)               |                          tcp_write_xmit() {
 0)   0.132 us    |                            tcp_tso_segs();
 0)   0.058 us    |                            tcp_init_tso_segs();
 0)               |                            tcp_transmit_skb() { /* 下发 skb ，交由网络层继续 */
 0)   0.046 us    |                              skb_push();
 0)               |                              ip_queue_xmit() {  /* 网络层，IP 协议的处理 */
 0)               |                                __sk_dst_check() {
 0)   0.069 us    |                                  ipv4_dst_check();
 0)   0.429 us    |                                }
 0)   0.049 us    |                                skb_push();
 0)               |                                  ip_output() {
 0)               |                                    ip_finish_output() {
 0)               |                                      ip_finish_output2() {
 0)   0.044 us    |                                        skb_push();
 0)               |                                        dev_queue_xmit() {   /* 数据链路层，由网卡设备（内核根据物理设备抽象出来的概念）继续处理 */
 0)               |                                          __dev_queue_xmit() {
 0)   0.151 us    |                                            netdev_pick_tx();
 0)   0.046 us    |                                            _raw_spin_lock();
 0)               |                                            /* some important things omitted */
 0) + 39.674 us   |                                          }
 0) + 40.053 us   |                                        }
 0) + 41.656 us   |                                      }
 0) + 43.041 us   |                                    }
 0) + 47.121 us   |                                  }
 0) + 68.614 us   |                                }
 0) + 70.558 us   |                              }
 0) + 75.995 us   |                            }
 0) + 85.718 us   |                          }
 0) + 86.418 us   |                        }
 0) + 87.275 us   |                      }
 0) ! 101.636 us  |                    }
 0) ! 105.024 us  |                  }
 0) ! 105.516 us  |                }
 0) ! 107.861 us  |              }
 0) ! 108.230 us  |            }
 0) ! 108.653 us  |          }
 0) ! 109.066 us  |        }
 0) ! 112.023 us  |      }
 0) ! 113.078 us  |    }
 0) ! 113.374 us  |  }
```

本篇核心关注流量控制，就不细数网络协议栈了。待到网络包(skb)到达数据链路层，`dev_queue_xmit` 标志着 skb 终于交付给网络设备，下面就是 TC 策略的运转了。

Qdisc (queueing discipline) ，是整个 TC 的基本模型。所有需要通过网卡接口发送的数据包，都会进入接口绑定的 Qdisc 等待队列(enqueue)。再由 ksoftirq 内核线程读取接口 Qdisc 中的数据包(dequeue)，尽最大能力发送出去。

## Traffic Control 原理

对于 TC 核心的概念——队列规程，类别，过滤器，网上充斥着大量的资料，由于没有自信讲得更加通透，附上链接一枚。

https://blog.csdn.net/dog250/article/details/40483627

## Traffic Control 实现

### 数据结构 

遵循 TC Qdisc 的生命周期，首先要介绍的是 Qdisc 的创建流程。不过在此之前，看了解下相关的数据结构。

![](https://img.ffutop.com/D4466735-ED98-4A94-9C7A-FD59F7791988.png)

```c
struct net_device
{
    /* 设备的发送队列列表 (这是数组的头指针)*/
    struct netdev_queue *_tx;

    /* 发送队列的个数 */
    unsigned int num_tx_queues;

    /* 使用中的发送队列个数 */
    unsigned int real_num_tx_queues;

    /* 默认的队列策略 */
    struct Qdisc *qdisc;

    /* 每个发送队列的最大长度 */
    unsigned long tx_queue_len;

    /* 发送队列的全局锁 */
    spinlock_t tx_global_lock;
}

struct netdev_queue {
    /* 所属网络设备 */
    struct net_device *dev;
    /* 队列相应的队列策略 */
    struct Qdisc *qdisc;
    struct Qdisc *qdisc_sleeping;
    spinlock_t _xmit_lock;
    int            xmit_lock_owner;
    unsigned long        trans_start;
    unsigned long        trans_timeout;
    /* 队列状态 */
    unsigned long        state;
};

#define TCQ_F_BUILTIN        1
#define TCQ_F_INGRESS        2
#define TCQ_F_CAN_BYPASS    4
#define TCQ_F_MQROOT        8
#define TCQ_F_ONETXQUEUE    0x10
#define TCQ_F_WARN_NONWC    (1 << 16)

struct Qdisc {
    /* skb 入队函数 */
    int (*enqueue)(struct sk_buff *skb, struct Qdisc *dev);
    /* skb 出队函数 */
    struct sk_buff * (*dequeue)(struct Qdisc *dev);
    unsigned int flags;
    struct list_head    list;
    int padded;
    /* 队列策略函数集 */
    const struct Qdisc_ops *ops;
    int            (*reshape_fail)(struct sk_buff *skb,
                    struct Qdisc *q);
    /* 所属设备队列 */
    struct netdev_queue *dev_queue;
    struct Qdisc *next_sched;

    /* 队列策略的状态 */
    unsigned long state;
    struct sk_buff_head q;
    u32 limit;
};
```

### 网络设备初始化与 Qdisc

TC 的实施与网络设备密切相关。跳过设备的注册和创建流程，设备状态由 `state DOWN` 切换到 `state UP` 标志着设备正式启用

![](https://img.ffutop.com/BF9E5268-8C81-499D-82B6-20DA7DF57473.png)

对应到内核流程，调用栈类似：

```c
dev_open
|-- __dev_open
    |-- dev_active
        |-- attach_default_qdiscs

static void attach_default_qdiscs(struct net_device *dev)
{
    struct netdev_queue *txq;
    struct Qdisc *qdisc;

    /* 获得设备的第 0 个 queue */
    txq = netdev_get_tx_queue(dev, 0);

    /* 如果发送队列个数 <= 1 || 发送队列长度 = 0 */
    if (!netif_is_multiqueue(dev) || dev->tx_queue_len == 0) {
        /* 单队列的流量控制 */
        netdev_for_each_tx_queue(dev, attach_one_default_qdisc, NULL);
        dev->qdisc = txq->qdisc_sleeping;
        atomic_inc(&dev->qdisc->refcnt);
    } else {
        /* 多队列的流量控制; 此处 mq 指 multiqueue*/
        qdisc = qdisc_create_dflt(txq, &mq_qdisc_ops, TC_H_ROOT);
        if (qdisc) {
            qdisc->ops->attach(qdisc);
            dev->qdisc = qdisc;
        }
    }
}

static void attach_one_default_qdisc(struct net_device *dev,
                     struct netdev_queue *dev_queue,
                     void *_unused)
{
    /* noqueue 默认的，无队列控制实现，一般应用在虚拟设备上 */
    struct Qdisc *qdisc = &noqueue_qdisc;

    /* 如果有发送队列长度限制 (默认应用在物理网卡设备上) */
    if (dev->tx_queue_len) {
        /* 创建 快速优先队列 策略的 Qdisc */
        qdisc = qdisc_create_dflt(dev_queue,
                      &pfifo_fast_ops, TC_H_ROOT);
    }
    dev_queue->qdisc_sleeping = qdisc;
}
```

### Qdisc 的生命周期

应用程序发送数据包都将涉及到系统调用，一般来说是 `sendmsg`、`sendto` 之类的系统调用。上面👆给了内核 `sendmsg` 的函数调用链路。很长，涉及的操作相当多。但是，这还仅仅是由 CPU 执行的一部分流程。试想，还有内存到网卡设备的数据交互。这里等待的时间将更加漫长，更不要说像是网络重试之类的实现了。内核究竟做了怎样的实现呢？在[这篇网络](https://www.ffutop.com/posts/2019-01-15-understand-kernel-8/)中，描述了接收网络数据时的三阶段流程。通过接收队列作为中介，`recv` 系统调用需要做的只是去检测接收队列是否有数据包，其它流程全部交由内核线程和网络硬中断完成。类似的，发送数据包 `sendmsg` 这类系统调用，也只是将数据包提交到一个等待队列，再由内核线程和中断实施尽力发送。这里的等待队列，就是网络设备绑定的 Qdisc。

先别管 Qdisc 是怎么实现的，它提供了两个核心函数 `enqueue`, `dequeue` ，完全符合定义一个队列的基本要求。

额外地，一些函数 `peek`, `reset` 等，作为这个黑盒的辅助函数，在个别情况下进行使用。

- peek: 类似 dequeue，但获取的网络包仍然保留在队列中

- reset: 将 Qdisc 重置到初始状态

- init: 初始化一个新创建的 Qdisc

- destroy: 销毁一个 Qdisc 生命周期中所使用的资源

- change: 修改 Qdisc 的参数

Qdisc 的整个生命周期，最初和最末分别是向内核注册一种新的 Qdisc 和取消注册一种 Qdisc

```c
/* from net/sched/sch_api.c */
int register_qdisc(struct Qdisc_ops *qops)
int unregister_qdisc(struct Qdisc_ops *qops)
```

再就是创建一个 Qdisc 实例绑定到网络设备，以及从网络设备上解绑 Qdisc 实例

```c
static struct Qdisc *
qdisc_create(struct net_device *dev, struct netdev_queue *dev_queue,
	     struct Qdisc *p, u32 parent, u32 handle,
	     struct nlattr **tca, int *errp)

// n->nlmsg_type == RTM_DELQDISC
static int tc_get_qdisc(struct sk_buff *skb, struct nlmsghdr *n)
```

再就是正常的工作流程了，`enqueue`, `dequeue` 以及其它辅助函数的使用。

### Qdisc 执行流

`noop` 是 Qdisc 最简单，最无赖的一种实现，对于所有的数据包直接抛弃。当然，这种实现正常情况下只会出现在网络设备状态为 DOWN 的时候。

```c
static int noop_enqueue(struct sk_buff *skb, struct Qdisc * qdisc)
{
    kfree_skb(skb);
    return NET_XMIT_CN;
}

static struct sk_buff *noop_dequeue(struct Qdisc * qdisc)
{
    return NULL;
}
```

`pfifo_fast` 是普遍使用一种 Qdisc 实现，根据数据包 Tos 将网络包划分到三个通道，进入第0通道的网络包有 dequeue 的最高优先级。

```c
static int pfifo_fast_enqueue(struct sk_buff *skb, struct Qdisc *qdisc)
{
    // 检测 Qdisc 队列数据包数量是否达到 dev 预定的最大值
    if (skb_queue_len(&qdisc->q) < qdisc_dev(qdisc)->tx_queue_len) {
        // 确定数据包需要进入哪个通道
        int band = prio2band[skb->priority & TC_PRIO_MAX];
        struct pfifo_fast_priv *priv = qdisc_priv(qdisc);
        // 获取通道列表的 head
        struct sk_buff_head *list = band2list(priv, band);

        priv->bitmap |= (1 << band);
        qdisc->q.qlen++;
        // 添加到通道队尾
        return __qdisc_enqueue_tail(skb, qdisc, list);
    }

    return qdisc_drop(skb, qdisc);
}

static struct sk_buff *pfifo_fast_dequeue(struct Qdisc *qdisc)
{
    struct pfifo_fast_priv *priv = qdisc_priv(qdisc);
    int band = bitmap2band[priv->bitmap];

    if (likely(band >= 0)) {
        struct sk_buff_head *list = band2list(priv, band);
        struct sk_buff *skb = __qdisc_dequeue_head(qdisc, list);

        qdisc->q.qlen--;
        if (skb_queue_empty(list))
            priv->bitmap &= ~(1 << band);

        return skb;
    }

    return NULL;
}
```

至于类别队列，首先也还是 Qdisc，符合它的所有要素，只是在 `Qdisc_ops` 中加入了分类的流程。例如 htb 的实现中，额外地加入了 `class_ops`，对关于分类操作的方法做了相关的实现。

```c
static struct Qdisc_ops htb_qdisc_ops __read_mostly = {
    .cl_ops    =    &htb_class_ops,
    .id        =    "htb",
    .priv_size =    sizeof(struct htb_sched),
    .enqueue   =    htb_enqueue,
    .dequeue   =    htb_dequeue,
    .peek      =    qdisc_peek_dequeued,
    .drop      =    htb_drop,
    .init      =    htb_init,
    .reset     =    htb_reset,
    .destroy   =    htb_destroy,
    .dump      =    htb_dump,
    .owner     =    THIS_MODULE,
};
static const struct Qdisc_class_ops htb_class_ops = {
    .graft       =    htb_graft,
    .leaf        =    htb_leaf,
    .qlen_notify =    htb_qlen_notify,
    .get         =    htb_get,
    .put         =    htb_put,
    .change      =    htb_change_class,
    .delete      =    htb_delete,
    .walk        =    htb_walk,
    .tcf_chain   =    htb_find_tcf,
    .bind_tcf    =    htb_bind_filter,
    .unbind_tcf  =    htb_unbind_filter,
    .dump        =    htb_dump_class,
    .dump_stats  =    htb_dump_class_stats,
}
```

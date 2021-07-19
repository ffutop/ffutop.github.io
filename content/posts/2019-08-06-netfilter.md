---
title: Netfilter 导览 - based on iptables
author: fangfeng
date: 2019-08-06
categories:
  - 技术
tags:
  - netfilter
  - iptables
  - Kernel
  - Linux
  - nat
cover: https://img.ffutop.com/C3EC0615-1B86-4452-8B68-7D78B63967F1.png
---

`Netfilter`，内核中用于管理网络数据包的重要网络模块；`iptables`，运行在用户空间，用于控制 `Netfilter` 模块的一种软件。需要注意的，`iptables` 只能控制 IPv4 数据包，对于 IPv6 数据包，需要使用 `ip6tables` 。

<!--more-->

## 表、链、规则

table、chain、rule 是 netfilter 中重要的三层概念，netfilter 将不同的功能划分成表(table)；每个表有若干规则链(chain)，其中有默认的规则链，也可以添加用户自定义的规则链；每条规则链又由若干规则组成。当然，这里关于 table 和 chain 的父子关系划分过于决定，事实上它们更像是两个度量层面的组合关系。

![](https://img.ffutop.com/4BF1C014-9447-4A45-AD1E-E819EA893CCD.png)

上图更详细地展示了四张表与系统默认规则链之间的组合关系。例如 raw 表只能关联到 PREROUTING 和 OUTPUT 两条链上。mangle 表与所有五条规则链都可以关联。当然，每个表都可以无限制地添加用户自定义的规则链。

### tables

netfilter 主要有 4 张表，分别致力于实现不同的功能。根据在同一个系统默认规则链上的执行顺序，分别是：

- raw：用于处理异常状况
- mangle：数据包的修改，用于实现服务质量和路由策略
- nat：网络地址转换
- filter：数据包的过滤，用于防火墙规则

### chains

系统默认规则链，其实就是内核在实现上留下的若干钩子方法，分别挂在内核网络处理流程的不同节点上。每条默认规则链的核心目标如下，请配合封面图食用：

INPUT链：处理输入数据包
OUTPUT链：处理输出数据包
FORWARD链：处理转发数据包
PREROUTING链：用于目标地址转换(DNAT)
POSTOUTING链：用于源地址转换(SNAT)

### rule

至于规则，挂载在规则链上，逐一按照预定动作执行

*以 iptables 工具的语法为例*

```sh
iptables -t filter -A INPUT -s 172.16.1.0/24 -p tcp --dport 22 -j ACCEPT
# -t filter 指向 filter 表添加一条规则
# -A INPUT [rule-specification] 指向 INPUT 链添加一条路由规则
# -s 172.16.1.0/24 表示允许这个网段的机器来连接
# -p tcp 表示允许 tcp 协议
# -j ACCEPT 表示接受这样的请求
```

`-j [target]` 表示这条规则的执行结果，预置的执行结果及描述包括：

- ACCEPT ：接收数据包
- DROP ：丢弃数据包
- RETURN ：返回上一层（调用方）规则链
- SNAT ：源地址转换
- DNAT ：目标地址转换
- MASQUERADE ：IP伪装(NAT)，用于ADSL
- LOG ：日志记录
- [chain\_name] ：跳转到目标规则链继续处理

## 内核实现

OK，建立了对 netfilter 的基本认知之后，来看看内核的具体实现。不过首先需要从内核的网络处理流程说起。

![](https://img.ffutop.com/DD0D7104-D2AD-48C7-AB5A-C4BF7D91451F.png)
<small><center>接收网络数据执行流</center></small>

看一下一次接收网络数据所经历的执行流，核心有三：

1. 网卡接收到数据（物理过程），发出网络中断；内核接收到中断信号，并进入中断处理流程；将网卡的数据拷贝到内核空间，建立 `sk_buff` 数据结构来存储数据；将 `sk_buff` 塞到 `input_pkt_queue` 队尾，同时发起 `NET_RX_SOFTIRQ` 软中断信号

2. 内核线程 `ksoftirqd` 接收到 `NET_RX_SOFTIRQ`，并进入处理流程；从 `input_pkt_queue` 获取一个队首的 `sk_buff` 数据；依据 TCP/IP 网络模型，从数据链路层开始，逐层将数据向上传递到传输层；将处理过的数据添加到相应的 `socket` 连接的接收队列队尾；如果 `input_pkt_queue` 还有更多数据，则继续处理，直到被任务调度打断。

3. 应用程序在需要获取网络数据时，调用 `recvfrom` 系统调用，由用户态陷入内核态；检查套接字的接收队列是否有数据；有数据则数据出队并拷贝到用户空间指定地址，并退出内核态；没有数据则等待，直到有接收队列有数据或等待超时。

就接收网络数据的核心流程而言，无需 netfilter 也是一个完整的闭环。而 netfilter 提供的功能，像是一个钩子(hook)，提供一定程度上修改 `sk_buff` 字节数据的能力（通过预定的策略）。从封面图可以看到，netfiler 绝大多数的处理流程都是挂载到网络层来实施的。

--- 

再提供一张 4.15.0 版本内核函数调用链（某一次调用 `sendmsg` 的部分调用栈，通过 tracefs 获得）

```
 0)               |        __vfs_write() {
 0)               |          new_sync_write() {
 0)               |            sock_write_iter() {
 0)               |              sock_sendmsg() {
 0)               |                inet_sendmsg() {
 0)               |                  tcp_sendmsg() {
 0)               |                    tcp_sendmsg_locked() {
 0)               |                      tcp_push() {
 0)               |                        __tcp_push_pending_frames() {
 0)               |                          tcp_write_xmit() {
 0)               |                            tcp_transmit_skb() {
 0)               |                              ip_queue_xmit() {
 0)   0.049 us    |                                skb_push();
 0)   0.113 us    |                                ip_copy_addrs();
 0)               |                                ip_local_out() {
 0)               |                                  __ip_local_out() {
 0)   0.059 us    |                                    ip_send_check();
 0)               |                                    nf_hook_slow() {
 0)   0.155 us    |                                      ipv4_conntrack_defrag [nf_defrag_ipv4]();
 0)               |                                      iptable_raw_hook [iptable_raw]() {
 0)               |                                        ipt_do_table [ip_tables]() {
 0)   0.047 us    |                                          __local_bh_enable_ip();
 0)   0.920 us    |                                        }
 0)   1.350 us    |                                      }
 0)               |                                      ipv4_conntrack_local [nf_conntrack_ipv4]() {
 0)               |                                        nf_conntrack_in [nf_conntrack]() {
 0)   0.067 us    |                                          ipv4_get_l4proto [nf_conntrack_ipv4]();
 0)   0.175 us    |                                          __nf_ct_l4proto_find [nf_conntrack]();
 0)   0.193 us    |                                          tcp_error [nf_conntrack]();
 0)               |                                          nf_ct_get_tuple [nf_conntrack]() {
 0)   0.052 us    |                                            ipv4_pkt_to_tuple [nf_conntrack_ipv4]();
 0)   0.075 us    |                                            tcp_pkt_to_tuple [nf_conntrack]();
 0)   0.774 us    |                                          }
 0)   0.089 us    |                                          hash_conntrack_raw [nf_conntrack]();
 0)   0.814 us    |                                          __nf_conntrack_find_get [nf_conntrack]();
 0)   0.048 us    |                                          tcp_get_timeouts [nf_conntrack]();
 0)               |                                          tcp_packet [nf_conntrack]() {
 0)   0.048 us    |                                            _raw_spin_lock_bh();
 0)               |                                            tcp_in_window [nf_conntrack]() {
 0)   0.046 us    |                                              nf_ct_seq_offset [nf_conntrack]();
 0)   0.634 us    |                                            }
 0)               |                                            _raw_spin_unlock_bh() {
 0)   0.048 us    |                                              __local_bh_enable_ip();
 0)   0.346 us    |                                            }
 0)   0.089 us    |                                            __nf_ct_refresh_acct [nf_conntrack]();
 0)   2.548 us    |                                          }
 0)   7.500 us    |                                        }
 0)   8.034 us    |                                      }
 0)               |                                      iptable_mangle_hook [iptable_mangle]() {
 0)               |                                        ipt_do_table [ip_tables]() {
 0)   0.048 us    |                                          __local_bh_enable_ip();
 0)   0.703 us    |                                        }
 0)   1.071 us    |                                      }
 0)               |                                      iptable_nat_ipv4_local_fn [iptable_nat]() {
 0)               |                                        nf_nat_ipv4_local_fn [nf_nat_ipv4]() {
 0)               |                                          nf_nat_ipv4_fn [nf_nat_ipv4]() {
 0)   0.052 us    |                                            nf_nat_packet [nf_nat]();
 0)   0.436 us    |                                          }
 0)   0.797 us    |                                        }
 0)   1.128 us    |                                      }
 0)               |                                      ip_vs_local_reply4 [ip_vs]() {
 0)               |                                        ip_vs_reply4 [ip_vs]() {
 0)   0.193 us    |                                          ip_vs_out [ip_vs]();
 0)   0.554 us    |                                        }
 0)   0.834 us    |                                      }
 0)               |                                      ip_vs_local_request4 [ip_vs]() {
 0)               |                                        ip_vs_remote_request4 [ip_vs]() {
 0)               |                                          ip_vs_in [ip_vs]() {
 0)   0.136 us    |                                            ip_vs_in.part.27 [ip_vs]();
 0)   0.507 us    |                                          }
 0)   0.813 us    |                                        }
 0)   1.094 us    |                                      }
 0)               |                                      iptable_filter_hook [iptable_filter]() {
 0)               |                                        ipt_do_table [ip_tables]() {
 0)   0.046 us    |                                          comment_mt [xt_comment]();
 0)   0.114 us    |                                          mark_mt [xt_mark]();
 0)   0.050 us    |                                          __local_bh_enable_ip();
 0)   2.166 us    |                                        }
 0)   2.444 us    |                                      }
 0) + 19.694 us   |                                    }
 0) + 20.495 us   |                                  }
 0)               |                                  ip_output() {
 0)               |                                    nf_hook_slow() {
 0)               |                                      iptable_mangle_hook [iptable_mangle]() {
 0)               |                                        ipt_do_table [ip_tables]() {
 0)   0.047 us    |                                          __local_bh_enable_ip();
 0)   0.345 us    |                                        }
 0)   0.633 us    |                                      }
 0)               |                                      iptable_nat_ipv4_out [iptable_nat]() {
 0)               |                                        nf_nat_ipv4_out [nf_nat_ipv4]() {
 0)               |                                          nf_nat_ipv4_fn [nf_nat_ipv4]() {
 0)   0.048 us    |                                            nf_nat_packet [nf_nat]();
 0)   0.360 us    |                                          }
 0)   0.688 us    |                                        }
 0)   0.976 us    |                                      }
 0)   0.062 us    |                                      ipv4_helper [nf_conntrack_ipv4]();
 0)               |                                      ipv4_confirm [nf_conntrack_ipv4]() {
 0)   0.080 us    |                                        nf_ct_deliver_cached_events [nf_conntrack]();
 0)   0.413 us    |                                      }
 0)   3.529 us    |                                    }
 0)               |                                    ip_finish_output() {
 0)   0.362 us    |                                      __cgroup_bpf_run_filter_skb();
 0)   0.078 us    |                                      ipv4_mtu();
 0)               |                                      ip_finish_output2() {
 0)               |                                          // some func
 0)               |                                      }
 0)   1.751 us    |                            }
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

`raw`, `mangle` 等表分别以 `iptable_raw_hook`, `iptable_mangle_hook`, `iptable_nat_ipv4_local_fn`, `iptable_filter_hook`, `iptable_nat_ipv4_out` 的钩子函数出现在系统调用 `sendmsg` 的调用栈中。（具体实现不表，有兴趣的可以直接看源码）

## More

本篇只是 netfilter 关于网络层处理流程的一个简述，封面图还可以看到 netfiler 在数据链路层的相关处理。另外，网络层协议也还包括 IPv6 ，netfilter 也是做了单独的实现的。

**参考**

- [netfilter project](https://netfilter.org/)
- Linux Kernel v4.15.0


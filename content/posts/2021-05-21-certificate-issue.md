---
title: ACME 协议下的域名证书签署
author: fangfeng
date: 2021-05-21
tags:
  - ACME
  - certificate
categories:
  - Cheat Sheet
---

## 背景

区别于私有化部署，用户对互联网上提供的在线服务，都希望能够建立在一个安全、可信赖的通信环境下。HTTPS 是用户普遍认知的一种传输安全协议，其本质是在 TCP/IP 协议栈的传输层(TCP)与应用层(HTTP)间增加 SSL/TLS 安全协议来保证通信安全。为了引入 SSL/TLS ，需要通过 CA 获取一份 SSL/TLS 证书。

![](//img.ffutop.com/6ca48149-0dd6-4046-8d54-de7e56b406b8.png)

[__图1 HTTP vs HTTPS__](https://www.cloudflare.com/zh-cn/learning/ssl/what-is-ssl/)

本文档描述了证书选型，DNS 服务商选型，证书签署及续签流程。

<!--more-->

## 证书选型

从签发验证的严格程度区分，证书分三类：

- DV SSL 证书（域名验证）：只验证域名所有权

- OV SSL 证书（组织验证）：验证域名所有权和申请组织的真实身份，可在证书详情里查看申请企业名称

- EV SSL 证书（扩展验证）：最高级别验证，验证域名所有权、企业真实合法信息、电话回访等等验证，可在浏览器地址栏显示绿色企业名称。

从支持的域名使用范围区分，也分三类：

- 单域名 SSL 证书：证书只能供单个特定域名使用

- 多域名 SSL 证书：证书只能供提前确定的有限个域名使用

- 通配符 SSL 证书：证书能供符合通配规则的所有域名使用

包括 DigiCert、GeoTrust、GlobalSign、Let's Encrypt 等 CA 均提供免费或付费的 SSL/TLS 证书的签发服务。

由于整个 SaaS 服务的部署成本受到严重限制，同时又考虑到未来维护的便捷性，决定采用 Let's Encrypt DV 通配符证书（免费，单次签发有效期 3 个月，可全程自动续签）。

[Let's Encrypt - 免费的SSL/TLS证书](https://letsencrypt.org/zh-cn/)

## DNS 服务商

DNS 服务商作为域名解析服务的提供方

- 为用户提供各种类型域名记录（比如 A、AAAA、CNAME、NS、SOA 等记录类型）的增删改查

- 向整个分布式域名系统提供特定域名的权威结果

在整个大域名系统中，有各种各样的域名服务商存在，甚至可以自建域名服务器提供自有域名的权威解析服务。为了能够准确地反映域名记录，标准提出了 NS 记录，来描述特定域名由哪个域名服务器提供权威解析。

为了未来操作管理流程的便捷，希望统一改写域名NS 记录，全部以阿里云作为 DNS  服务商（与 SaaS 服务基础设施提供方保持一致）。

```text
ffutop.com IN NS ns1.alidns.com ns2.alidns.com
```

在完成修改后，由于 DNS 是一个大型分布式系统，由大量服务器组成且各自有它的缓存时间，故整个生效时间可能长达几天。为了快速确认是否完成修改，可以直接查看注册局的资料。

```shell
[ffutop@desktop ~] $ whois ffutop.com | grep 'Name Server'
Name Server: ns1.alidns.com
Name Server: ns2.alidns.com
```

## 证书签署

Let's Encrypt 使用 ACME 协议签署证书。签署通配符域名主体流程分为两步

1. ACME 管理软件向证书颁发机构证明拥有对域名的控制权。

1. 该管理软件申请、续期或吊销该域名的证书。 

[https://github.com/acmesh-official/acme.sh/wiki/%E8%AF%B4%E6%98%8E](https://github.com/acmesh-official/acme.sh/wiki/%E8%AF%B4%E6%98%8E)

### 安装 ACME 管理软件

由于国内的 DNS 服务商对 certbot （ Let's Encrypt 强烈推荐的 ACME 工具）的支持度很低。故选择使用 acme.sh （另一种 ACME 工具）

[__https://github.com/acmesh-official/acme.sh__](https://github.com/acmesh-official/acme.sh)

```shell
# 自动下载并安装 acme.sh
# 其中 email=my@example.com 用于向 CA 注册邮箱地址，用于接收证书即将到期的提醒邮件
curl https://get.acme.sh | sh -s email=my@example.com

# acme.sh 将被安装到当前用户的 ~/.acme.sh/
# 可以添加 alias acme.sh=~/.acme.sh/acme.sh 便捷使用
```

### 申请签署 SSL/TLS 证书

下面以 ffutop.com 为例介绍整个流程。

为了整个证书申请流程的自动化，需要直接操作 DNS 服务商提供的 API。我们使用阿里云 DNS，需要基于管理域名的阿里云账号构建一套 AK/SK (@See [__https://usercenter.console.aliyun.com/#/manage/ak__](https://usercenter.console.aliyun.com/#/manage/ak))。

```shell
# DNS 服务商为 alidns，如果服务商改了，请查阅文档并修改 https://github.com/acmesh-official/acme.sh/wiki/dnsapi
# Ali_Key, Ali_Secret 在本次操作后会被保存到 ~/.acme.sh/account.conf 供下次使用
export Ali_Key="xxxxxxx"
export Ali_Secret="yyyyyyy"
acme.sh --issue --dns dns_ali -d *.ffutop.com
```

### 配置并使用 SSL/TLS 证书

在现有架构中，所有服务入口均由 NGINX 承接服务。故需要 NGINX 配置 SSL/TLS 证书。acme.sh 工具同时也提供了便捷的命令。

```shell
acme.sh --install-cert -d *.ffutop.com \
--key-file       /etc/nginx/ssl/ffutop.com.key.pem  \
--fullchain-file /etc/nginx/ssl/ffutop.com.cert.pem \
--reloadcmd     "nginx -s reload"
```

## 证书续签

acme.sh 工具安装时自动写入了一条定时任务

```text
[root@saas-nginx-1 ~] $ crontab -l
49 0 * * * "/root/.acme.sh"/acme.sh --cron --home "/root/.acme.sh" > /dev/null
```

定时任务将每日运行，并在 60 日时自动更新证书（并为 NGINX 配置新的证书）。整个过程都是自动的，无需人工干预。

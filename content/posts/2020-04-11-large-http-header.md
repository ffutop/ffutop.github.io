---
title: HTTP Large Header Fields Problem
author: fangfeng
date: 2020-04-11
categories:
  - 技术
tags:
  - NGINX
  - Tomcat
  - Java
  - HTTP
---

*首次遇到请求头过大的问题，做个记录。特别是在本次处理陷入了误区，做了太多无谓的猜测*

请求头过大导致响应错误码 400 (Bad Request)、414 (URI Too Long)、431 (Request Header Fields Too Large) 的情况不多，不过原因和解决方案都是比较清晰的。客户端请求的请求头过大导致超出了服务器支持的缓冲区。如果客户端可控，控制请求头的大小；否则，适当调大服务器配置的缓冲区大小。

最近生产上碰到了这个问题，颇费了一番功夫。接手问题时得到了几个错误的信息，干扰到了处理的全过程。甚至为此去重读了 NGINX Directive `client_header_buffer_size` 和 `large_client_header_buffers` 在 1.8.1 版本的实现。

最原始的问题是：NGINX 接收到了大请求头(4.5k)的请求，最终响应了错误码 400 Bad Request 。

真实的背景因素包括：

- 请求链路 NGINX -> k8s nginx ingress -> k8s pods (Tomcat) 
- NGINX `large_client_header_buffers` 使用了默认配置 `4 8k`。
- Tomcat maxHttpHeaderSize 使用了默认配置 (default 8192)

<!--more-->

## 背景

**HTTP Request message syntax**

```plain
Request = Request-Line
        *(( general-header
        | request-header
        | entity-header ) CRLF)
        CRLF
        [ message-body ]

Request-Line = Method SP Request-URI SP HTTP-Version CRLF
```

**HTTP Response message syntax**

```plain
Response = Status-Line
         *(( general-header
         | response-header
         | entity-header ) CRLF)
         CRLF
         [ message-body ]
Status-Line = HTTP-Version SP Status-Code SP Reason-Phrase CRLF
```

**Related Response Status Code**

- 400 Bad Request
- 414 URI Too Long
- 431 Request Header Fields Too Large
    - Total size of request headers too large
    - Or, a single header field is too large

**NGINX Configuration**

Nginx 与请求头缓冲区相关的指令有两个：`client_header_buffer_size` 和 `large_client_header_buffers` 。

![Nginx 处理请求头逻辑](https://img.ffutop.com/2A3B45A8-72BB-4D6C-9C46-0126C09FE157.png)

`client_header_buffer_size` (default 1k)

定义用于读取请求头的缓冲区大小。如果请求头过大，将依据 `large_client_header_buffers` 指令临时申请大缓冲区来处理。

`large_client_header_buffers` (default 4 8k)

用来处理偶尔出现的过大的请求头，该缓冲区创建后，先拷贝原缓冲区读取的内容，然后继续读取未读完的内容。此类缓冲区力求达到的是临时申请，迅速释放的目的。

- 如果 request line 过大，超出缓冲区大小，NGINX 将响应 414 错误码

- 如果 request header field 过大，NGINX 将相应 400 错误码（事实上，个人认为用 431 错误码更合适）

## 问题原因

重新梳理过 NGINX、Tomcat 配置之后，它们对请求头的极限大小应该是 8k 。按理说 4.5k 的请求头无论如何都不会触发任何的问题。而且，即使 NGINX 反向代理增加了类似 X-Forwarded-For 之类的来源描述，整个请求头不会超过 5k 。

但是，也正是这个错误信息影响了整个处理流程。直接到 Tomcat 所在的 Kubernetes Pod 里面抓个包，马上就能明白，被 NGINX 额外添加的远远不止想象的这么点东西。

`X-Original-Uri` 将整个请求头大小做了倍乘。Tomcat 收到了大约 9k 的请求头，从而也就直接导致了 400 错误码的出现。

那么，这个 `X-Original-Uri` 又是怎么被引入的呢？查了 [ingress-nginx](https://github.com/kubernetes/ingress-nginx) 的仓库: 

- 2017 年 3 月，`proxy_pass X-Original-URI` 首次出现，具体原因没有找到相应的 Issues (怀疑仓库迁移到 GitHub 发生在 2017 年中)，不过从当时 commit 信息来看，大约是为了 oauth 而引入的

- 2018 年 8 月，[Issue #2353](https://github.com/kubernetes/ingress-nginx/pull/2353) 为 `X-Original-Uri` 提供了一个可选的开关 (`proxy-add-original-uri-header`)。但为了兼容性考虑，默认值被配置为开启

- 2019 年 9 月，[Issue #4604](https://github.com/kubernetes/ingress-nginx/pull/4604) 调整了开关，将其默认值配置为关闭。此 Issue 产生的原因，也正是 `X-Original-Uri` 引起了后端服务器 Code 431 的错误响应。

## 解决方案

1. 由于目前使用的 nginx-ingress 版本存在选项 `proxy-add-original-uri-header` 。直接将其置为关闭即可。

2. 升级 nginx-ingress，不过这个动作就不得不进行兼容性测试了。


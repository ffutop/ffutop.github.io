---
title: NGINX 的域名解析缓存
author: fangfeng
date: 2021-04-03
tags:
    - Nginx
    - DNS
    - Cache
    - TTL
categories:
    - 案例分析
---

NGINX 作为常用的反向代理软件，普遍适用于对外统一暴露各种内部服务等常见。在使用过程中，常使用内网 IP 直接声明需要反向代理的服务。

恰恰因为常规的使用方式都是配置上游(Upstream)服务的 IP，偶然间遇到使用域名配置上游服务，竟然陷入了知识盲区。

## 异常

为了对 HTTP 响应头做一些额外的处理，我们用 NGINX 代理了自建的图片 CDN 服务。精简后的配置如下

```plain
server {

    listen 443 ssl;
    server_name proxy.ffutop.com;

    ssl_certificate     ffutop.com.rsa.crt;
    ssl_certificate_key ffutop.com.rsa.key;

    location /proxyimg/ {
        proxy_set_header Host img.ffutop.com;
        proxy_pass https://img.ffutop.com;
        include extra_ops.conf;
    }
}
```

平稳运行一段时间之后，突然所有指向 `proxy.ffutop.com/proxyimg/XXX/YYY` 的请求全部响应为 `502 Bad Request` 。

为了排查这个问题，收集到了如下信息：

1. 越过 NGINX 代理，直接请求图片 CDN，服务正常响应。
2. 确认命中了前述配置，NGINX 配置未发生修改。
3. NGINX 日志记录了相关请求的 Upstream 全都指向了 *A* IP （此问题发生前后都相同）
4. 检测 NGINX 宿主机的域名解析结果(`dig img.ffutop.com`)，IP 为 *B*

## 排查

基于收集到的信息，确认是域名解析发生问题，但 NGINX 的域名解析直接依赖宿主机 `/etc/resolv.conf` 配置声明的 DNS 服务器，机器上确认了域名解析的记录没有问题。结合语言及软件普遍存在对 DNS TTL 的非标准化处理，故判断 NGINX 也存在这样的问题。

> NGINX caches the DNS records until the next restart or configuration reload, ignoring the records’ TTL values.[1]

翻阅资料确认了这个问题，NGINX 将无视 TTL 并缓存 DNS 记录，直到下一次重启或配置重载。至于域名解析，NGINX 启动时就对 `img.ffutop.com` 进行了解析。

## 解决

临时地，为了快速恢复，直接将 NGINX 服务做了配置重载。

但 NGINX 对 DNS TTL 的非标实现，对 IP 频繁发生变更的服务是无法接受的。

如何解决这个问题？NGINX 确实提供了标准实现，通过提供 `resolver` 指令声明 DNS 服务器地址，NGINX 将在 DNS 记录 TTL 到期后，重新解析域名。归根到底是自己的配置问题啊 T\_T

```plain
resolver 223.5.5.5;

server {
    location / {
        set $backend_servers img.ffutop.com;
        proxy_pass http://$backend_servers;
    }
}
```

## 参考资料

\[1\]. [Using DNS for Service Discovery with NGINX and NGINX Plus](https://www.nginx.com/blog/dns-service-discovery-nginx-plus/)


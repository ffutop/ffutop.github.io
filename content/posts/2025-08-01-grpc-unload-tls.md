---
title: gRPC mTLS 问题调试指南
author: ffutop
date: 2025-08-01
tags:
  - gRPC
  - mTLS
  - TLS
  - Debugging
categories:
  - Cheat Sheet
---

## 前言

在分布式系统中，gRPC 和 mTLS（双向 TLS）是保障服务间通信安全的黄金搭档。然而，当 TLS 握手失败或行为不符合预期时，问题排查可能会变得非常棘手。

本篇旨在为你提供一个快速、高效的排查指南，帮助你迅速定位并解决常见的 gRPC mTLS 问题。

## TLS 通信数据脱密

当你需要深入了解 TLS 握手细节或检查应用层数据时，解密 gRPC 流量是必不可少的步骤。**需要明确的是，直接使用 Wireshark 抓包，你看到的只会是无法阅读的 TLS 加密报文。** 为了看到内部的 gRPC 通信内容，我们必须对流量进行解密。

目前主流的解密方法有两种：

1. **使用 Pre-Master Secret (推荐)**: 这是最通用和现代的方法，支持包括 `TLS 1.3` 和 `PFS` (Perfect Forward Secrecy) 在内的所有密码套件。
2. **使用 RSA 私钥**: 这种方法较为传统，**仅在 TLS 握手使用 RSA 密钥交换算法时有效**。如果使用了 `DHE` 或 `ECDHE` 等具有前向保密性的算法，此方法将失效。

本指南主要介绍第一种方法。

### 方法一：使用 Pre-Master Secret

#### 核心思路

通过 `jSSLKeyLog` 这个 Java Agent，我们可以捕获 TLS `(Pre)-Master Secret`，并利用 Wireshark 等工具解密 TLS 报文。

> **注意**: 此方法不支持 BoringSSL，因此需要将 gRPC 配置为使用 `SunJSSE`。

#### 操作步骤清单

1. **下载 [jSSLKeyLog](https://jsslkeylog.github.io/)**：获取最新的 `jSSLKeyLog.jar` 文件。
2. **配置 SSL Provider**：修改你的 gRPC 客户端/服务端代码，强制使用 `SunJSSE` 作为 `Provider`。
3. **配置 JVM 启动项**：在启动你的 Java 应用时，添加 `javaagent` 参数。
    ```bash
    java -javaagent:path/to/jSSLKeyLog.jar=/path/to/your_logfile.log -jar your-app.jar
    ```
4. **分析流量**：使用 Wireshark 打开抓包文件，并在 `Preferences -> Protocols -> TLS` 中配置 `(Pre)-Master-Secret log filename`，指向你生成的 `your_logfile.log`。

#### 客户端配置示例

为了让客户端使用 `SunJSSE`，你需要通过 `GrpcSslContexts.configure()` 方法指定 `Provider`。

```java
@Bean
public GrpcChannelConfigurer clientConfigurer() throws SSLException {
    // 1. 构建 SslContext，指定 CA 证书用于验证服务端
    SslContextBuilder sslClientContextBuilder = GrpcSslContexts.forClient()
            .trustManager(new File("classpath:tls/ca.crt"));
    sslClientContextBuilder.protocols("TLSv1.2"); // 建议指定协议版本

    // 2. 关键步骤：配置使用 SunJSSE Provider
    SslContext clientSslCtx = GrpcSslContexts.configure(sslClientContextBuilder,
            Security.getProvider("SunJSSE")).build();

    // 3. 将 SslContext 应用到 ChannelBuilder
    return (channelBuilder, name) -> {
        if (channelBuilder instanceof NettyChannelBuilder) {
            ((NettyChannelBuilder) channelBuilder)
                    .sslContext(clientSslCtx)
                    .defaultServiceConfig(loadServiceConfig())
                    .keepAliveTime(30, TimeUnit.SECONDS)
                    .keepAliveTimeout(5, TimeUnit.SECONDS);
        }
    };
}
```

#### 服务端配置示例

服务端配置与客户端类似，同样需要指定 `Provider`。

```java
@Bean
public GrpcServerConfigurer grpcServerConfigurer() throws SSLException {
    // 1. 构建 SslContext，提供服务端证书和私钥
    SslContextBuilder sslServerContextBuilder = GrpcSslContexts.forServer(
            new File("classpath:tls/tls.crt"), 
            new File("classpath:tls/tls.key")
    );
    sslServerContextBuilder.protocols("TLSv1.2");

    // 2. 关键步骤：配置使用 SunJSSE Provider
    SslContext serverSslCtx = GrpcSslContexts.configure(sslServerContextBuilder,
            Security.getProvider("SunJSSE")).build();

    // 3. 将 SslContext 应用到 ServerBuilder
    return serverBuilder -> {
        if (serverBuilder instanceof NettyServerBuilder) {
            ((NettyServerBuilder) serverBuilder)
                    .sslContext(serverSslCtx)
                    .permitKeepAliveTime(30, TimeUnit.SECONDS);
        }
        serverBuilder.intercept(TransmitStatusRuntimeExceptionInterceptor.instance());
    };
}
```

### 方法二：使用 RSA 私钥

如果你的 TLS 握手确定使用了 RSA 密钥交换，你也可以通过在 Wireshark 中配置服务端的私钥来解密流量。

关于如何在 Wireshark 中配置 RSA Keys 列表，请参考这篇详细指南：
[基于 RSA 密钥交换算法的 TLS 数据包解析](https://www.ffutop.com/posts/2023-03-18-tls-parse/#%E5%9F%BA%E4%BA%8E-rsa-%E5%AF%86%E9%92%A5%E4%BA%A4%E6%8D%A2%E7%AE%97%E6%B3%95%E7%9A%84-tls-%E6%95%B0%E6%8D%AE%E5%8C%85%E8%A7%A3%E6%9E%90)

### 延伸阅读

- [TLS 加密报文解析](https://www.ffutop.com/posts/2023-03-18-tls-parse/)
- [如何在 gRPC 中导出 TLS Master Keys](https://wiki.wireshark.org/How-to-Export-TLS-Master-keys-of-gRPC)

## 常见故障排查清单

请按以下顺序检查你的配置：

### 1. 服务端 `SslContext` 配置

- 检查点：确保服务端 `SslContext` 不仅加载了自己的证书和私钥，还配置了用于验证客户端证书的 `TrustManager`（即 CA 证书）。
- 示例：
  ```java
  SslContextBuilder.forServer(serverCert, serverKey)
    .trustManager(caCert) // <--- 是否配置了信任的 CA？
    .clientAuth(ClientAuth.REQUIRE) // <--- 是否要求客户端认证？
    .build();
  ```

### 2. 服务端 `ClientAuth` 模式

- 检查点：确保你明确要求客户端进行身份验证。
- `ClientAuth.REQUIRE` vs `ClientAuth.OPTIONAL`：
    - `REQUIRE` (对应 `setNeedClientAuth(true)`): 强制要求客户端提供证书。如果客户端不提供，握手将失败。这是标准 mTLS 的要求。
    - `OPTIONAL` (对应 `setWantClientAuth(true)`): 请求客户端提供证书，但即使不提供，握手也可以继续。
- 修复建议：在服务端的 `SslContextBuilder` 中，明确调用 `.clientAuth(ClientAuth.REQUIRE)`。

### 3. 客户端 `SslContext` 配置

- 检查点：确保客户端 `SslContext` 正确加载了它自己的证书 (`client.crt`) 和私钥 (`client.key`)。
- 示例：
  ```java
  SslContextBuilder.forClient()
    .trustManager(caCert) // 用于验证服务端
    .keyManager(clientCert, clientKey) // <--- 客户端是否提供了自己的证书和私钥？
    .build();
  ```

### 4. 客户端 SSL/TLS 参数与主机名验证

- 检查点：客户端除了提供证书外，还必须正确验证它所连接的服务端的身份。这通常通过主机名验证来完成。
- 修复建议：在 gRPC 客户端，确保你连接的 `authority` (主机名) 与服务端证书中的 `Common Name (CN)` 或 `Subject Alternative Name (SAN)` 匹配。如果主机名不匹配，即使证书本身是可信的，连接也会失败，并可能抛出 `SSLPeerUnverifiedException`。在使用 `NettyChannelBuilder` 时，可以通过 `.overrideAuthority()` 来强制指定用于验证的主机名。

### 5. 证书链完整性与有效性

- 检查点：
  - 客户端提供的证书是否由服务端信任的 CA 签发？
  - 如果使用了中间证书（Intermediate CA），确保证书链是完整的。客户端应发送完整的证书链（叶证书 + 中间证书），或者服务端信任存储中包含所有必要的中间证书。
  - 所有证书（CA、服务端、客户端）是否在有效期内？

### 6. 密码套件（Cipher Suites）兼容性

- 检查点：虽然不常见，但要确认客户端和服务端之间至少有一个共同支持的、安全的密码套件。不匹配的密码套件会导致握手失败。通常，gRPC 的默认配置已经足够好，但自定义配置时需要注意。

---

mTLS 的配置虽然细节繁多，但只要了解脱密原理，并遵循本文的排查清单，系统地检查每一个环节，绝大多数问题都能迎刃而解。希望这份指南能为你节省宝贵的调试时间，让你更专注于业务逻辑的实现。
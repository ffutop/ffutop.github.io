---
title: URI不规范编码解决方案
author: fangfeng
date: 2020-07-25
tags:
  - URI
  - precent-encoding
categories:
  - 技术
---

[RFC 7230](https://tools.ietf.org/rfc/rfc7230.txt) 与 [RFC 3986](https://tools.ietf.org/rfc/rfc3986.txt) 定义了 HTTP/1.1 标准并对 URI 的编解码问题作出了规范。但是，文本形式的规范和最终落地的标准之间总是存在着差距。标准中共 82 个字符无需编码。

```
ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789:/?#@!$&'()*+,;_-.~
```

对于需要编码的字符，百分号编码方案要求 ASCII 字符表示成两个 16 进制值并添加前缀 `%`；对于非 ASCII 字符, 需要转换为 UTF-8 字节序, 然后每个字节按照上述方式表示成两个 16 进制值并添加前缀 `%`。比如 `|` 将被表示为 `%7C` 或 `%7c`（大小写无关）；`啊` 将被表示为 `%E5%95%8A`。

最近因为 Tomcat 版本升级，遭遇了非标到标准实现的过渡，新版本 Tomcat 严格执行 RFC 规范直接将非标准的 URI 请求强制拒绝。

```plain
2020-07-15 16:02:12,931  INFO 1 --- [http-nio-8080-exec-7] o.a.c.h.Http11Processor                  : Error parsing HTTP request header
 Note: further occurrences of HTTP request parsing errors will be logged at DEBUG level.
java.lang.IllegalArgumentException: Invalid character found in the request target. The valid characters are defined in RFC 7230 and RFC 3986
        at org.apache.coyote.http11.Http11InputBuffer.parseRequestLine(Http11InputBuffer.java:468) ~[tomcat-embed-core-9.0.22.jar!/:9.0.22]
        at org.apache.coyote.http11.Http11Processor.service(Http11Processor.java:294) [tomcat-embed-core-9.0.22.jar!/:9.0.22]
        at org.apache.coyote.AbstractProcessorLight.process(AbstractProcessorLight.java:66) [tomcat-embed-core-9.0.22.jar!/:9.0.22]
        at org.apache.coyote.AbstractProtocol$ConnectionHandler.process(AbstractProtocol.java:853) [tomcat-embed-core-9.0.22.jar!/:9.0.22]
        at org.apache.tomcat.util.net.NioEndpoint$SocketProcessor.doRun(NioEndpoint.java:1587) [tomcat-embed-core-9.0.22.jar!/:9.0.22]
        at org.apache.tomcat.util.net.SocketProcessorBase.run(SocketProcessorBase.java:49) [tomcat-embed-core-9.0.22.jar!/:9.0.22]
        at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1149) [?:1.8.0_161]
        at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:624) [?:1.8.0_161]
        at org.apache.tomcat.util.threads.TaskThread$WrappingRunnable.run(TaskThread.java:61) [tomcat-embed-core-9.0.22.jar!/:9.0.22]
        at java.lang.Thread.run(Thread.java:748) [?:1.8.0_161]
```

<!--more-->

## Tomcat 侧解决方案

由于是 Tomcat 升级带来的问题，回退版本是最简单但无奈的选择。但这不是理想的解决方案。

Tomcat 为了兼容非标准实现的浏览器请求编码，额外释出了两个配置选项 `RelaxedPathChars` / `relaxedQueryChars` 分别用于对 URI Paths 和 URI Query 放松限制。通过配置 `server.xml` 的 `Connector` 允许自定义放过 `" < > [ \ ] ^ \` { | }` 中的一种或多种字符。

```xml
<Connector 
    connectionTimeout="20000" 
    port="8080" 
    protocol="HTTP/1.1" 
    redirectPort="8443" 
    relaxedQueryChars='^{}[]|' 
    relaxedPathChars='{}'
/>
```

至于 Spring Boot 内嵌的 Tomcat 容器，可以通过主动声明 Web 服务器工厂实例来配置

```java
@Bean
public ConfigurableServletWebServerFactory webServerFactory() {
    TomcatServletWebServerFactory factory = new TomcatServletWebServerFactory();
    factory.addConnectorCustomizers(connector -> {
        connector.setProperty("relaxedPathChars", "[]|");       // 添加需要的特殊符号
        connector.setProperty("relaxedQueryChars", "|{}[]");    // 添加需要的特殊符号
    });
    return factory;
}
```

## 反向代理服务器 NGINX 侧解决方案

Tomcat 在新版本中限制部分未编码字符的使用，是为了解决 HTTP 请求走私漏洞。放松对此类字符的限制无疑是主动交出大门钥匙。

浏览器的非标准实现方案无法规避。但对于我们现有的部署架构，NGINX 作为最外侧的服务，提供反向代理。在 NGINX 侧可以将非标 HTTP 请求标准化。

由于 NGINX rewrite 模块对特定字符全量替换支持度不足，引入了 Lua 模块来实现对特定字符 `|` 的编码处理，其他字符也可采用类似的方式来解决。

```plain
location / {
    rewrite_by_lua_block {
        local escape_args = string.gsub(ngx.var.args, "|", "%%7C")
        ngx.req.set_uri_args(escape_args)
    }

    proxy_pass http://k8s;
}
```


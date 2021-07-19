---
title: 劫持 Java 应用 HTTP 请求
author: fangfeng
date: 2020-10-19
tags:
  - HTTP
  - Hijack
  - Java
categories:
  - 技术
---

## 背景

全链路追踪中，针对部分特殊的流量，希望将它引导到特定服务上（这个特定服务不在正常请求的链路上）——问题可以被抽象为解决进程间通信过程中目标进程的选择。

进程间通信方式很多，本篇只关注 Java 进程间套接字通信下 HTTP 形式的请求劫持，引导特定流量到特定进程。

<!--more-->

## 解决方案

可行的处理方案繁多。自顶向下从应用、框架、JVM、Container Runtime、System Call、网络协议栈等级别，均可着手解决。侵入性最强的操作就是要求所有业务应用都主动实现 HTTP 请求分流逻辑；次一级是提供二方库供业务应用主动集成；或者从系统层面进行改造，基于改写系统调用对请求进行劫持。

回顾两年前的所学，JVM TI 为劫持 HTTP 请求提供了一个全新的解决思路。通过 Agent 改写应用启动时加载的类的字节码，劫持类的实例并完成目标功能。

由于 Java 项目间调用大量的使用了 Apache 的 http-client 库，改写变得相当简单。识别流量，并对特定流量改写请求的 Host 即可。

## Demo

由于 http-client 对所有请求目标都统一由 `org.apache.http.HttpHost` 维护，控制变得极为简单。只需在 `HttpHost` 实例化时，改写类的构造方法，即完成了对目标的劫持工作（下例中强制将所有请求指向 `163.com`）

```java
public class Agent implements ClassFileTransformer {

    public static void premain(String args, Instrumentation instrumentation) {
        instrumentation.addTransformer(new Agent());
    }

    @Override
    public byte[] transform(ClassLoader loader, String className, Class<?> classBeingRedefined, ProtectionDomain protectionDomain, byte[] classfileBuffer) throws IllegalClassFormatException {
        if ("org/apache/http/HttpHost".equals(className)) {
            ClassPool pool = ClassPool.getDefault();
            try {
                CtClass httpHost = pool.get("org.apache.http.HttpHost");
                CtClass string = pool.get("java.lang.String");
                CtConstructor constructor = httpHost.getDeclaredConstructor(new CtClass[]{string, CtClass.intType, string});
                constructor.insertBefore("hostname = \"www.163.com\";");
                byte[] bytes = httpHost.toBytecode();
                FileOutputStream fos = new FileOutputStream("/Users/fangfeng/a.class");
                fos.write(bytes);
                return bytes;
            } catch(NotFoundException | CannotCompileException | IOException e){
                e.printStackTrace();
            }
        }
        return classfileBuffer;
    }
}
```

将整个项目打包为 agent.jar 的过程不做太多介绍，详见 [ffutop/http-client-plugin](https://github.com/ffutop/http-client-plugin)

针对需要劫持的项目，在启动参数中增加 `-javaagent:${PATH_TO}/http-client-plugin.jar`



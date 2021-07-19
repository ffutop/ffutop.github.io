---
title: 如何方便地获取 CGlib 生成类
author: fangfeng
date: 2018-07-13
categories:
  - 技术
tags:
  - CGlib
  - tools
---

## 配置参数

**命令行使用**

在 java 启动命令中添加参数配置项 `-Dcglib.debugLocation=<Custom Path>`

**编码实现**

在执行 CGlib 获取新生成类之前，调用 `System.setProperty("cglib.debugLocation", <Custom Path>)`

<!--more-->

## 如何游刃有余地 Debug 掺杂 CGlib 生成类的调用链

经常会遇到，项目中不经意间就遇到了 CGlib 代理，然后对调用链的 Debug 就开始出现断层。继而，对对象或实例变量的跟踪就显得一头雾水。

特别是在不了解生成类到底是增强了那些内容的时候，甚至可能连调用链都跟踪不下去了 T\_T 。

通过 **配置参数** 一节的内容，你就可以在你理想的目录 `<Custom Path>` 下看到**所谓黑盒**中生成类的完成内容了。

下面的内容，将简单地介绍如何配合 CGlib 生成类的内容实现: 保证 Debug 的每一步骤可见; 保证 Debug 的过程不会出现*黑盒*。

### 确定 `<Custom Path>` 下哪个文件是原有 Java 类的生成类

```java
public String getClassName(String prefix, String source, Object key, Predicate names) {
    if (prefix == null) {
        prefix = "net.sf.cglib.empty.Object";
    } else if (prefix.startsWith("java")) {
        prefix = "$" + prefix;
    }
    String base =
        prefix + "$$" +
        source.substring(source.lastIndexOf('.') + 1) +
        getTag() + "$$" +
        Integer.toHexString(STRESS_HASH_CODE ? 0 : key.hashCode());
    String attempt = base;
    int index = 2;
    while (names.evaluate(attempt))
        attempt = base + "_" + index++;
    return attempt;
}
```

CGlib 默认的类名生成策略下，生成类的全限定名将结合原文件的全限定名决定。

通常命名如下: `<原 Java 类全限定名>$$<类似 EnhancerByCGlib>$$<生成类核心内容的 hash 值>_<index[可能存在]>`

例如: 原有 Java 类全限定名为 me.fangfeng.Test ，则生成类名可能为 `me.fangfeng.Test$$EnhancerByCGlib$$ab1203aa`

当然，通常情况下会存在一个类名形如 `me.fangfeng.Test$$FastClassBySpring$$...` 的类，这是作为生成类 `me.fangfeng.Test$$EnhancerByCGlib$$ab1203aa` 的辅助类来使用。

### 简单了解生成类下的调用关系

Java 代码触发被增强的类，将首先调用生成类。但是，这是没办法进行 Debug 的，因为这个生成类的字节码缺少 SourceFile 属性，同时缺少 LineNumberTable 属性。

但是，这并不影响对调用链的跟踪。

CGlib 的增强，所有的增强逻辑都是以 Callback 实现类的形式来提供的。同时在 CGlib 生成类中也有 Callback 的实例变量来持有这些注入的 Callback 。

**原有 Java 类**

![LogonService.java](https://ws3.sinaimg.cn/large/006tKfTcgy1ft8e7kr7qvj30sc0j0wh4.jpg)

**对应的生成类**

![](https://ws1.sinaimg.cn/large/006tKfTcgy1ft8e9ww4hqj31hg0oojzv.jpg)

![](https://ws1.sinaimg.cn/large/006tKfTcgy1ft8eanb6c9j31j00lgn1e.jpg)

**跟踪 LogonService.addLogon(String var1) 演示**

当前 LogonService 的生成类全限定名为me.fangfeng.transactionBugs.LogonService$$EnhancerBySpringCGLIB$$bfc1dc3.class; 以下简称为 LogonSerivce$$bfc1dc3 

1. 首先，配置 `cglib.debugLocation` 参数，值 = 项目生成的 .class 路径

2. 无断点直接运行一次需要处理的逻辑

3. 找到 LogonSerive$$bfc1dc3.class, 随意打上一个看似有效的断点

4. 在各处打上必要的断点，开始进行真正的调试工作

5. 代码执行到 LogonService$$bfc1dc3.class, 虽然没有真正进入断点位置, 但是可以看到这个 LogonService$$bfc1dc3 实例的实例变量信息

![](https://ws3.sinaimg.cn/large/006tKfTcgy1ft8eq1be61j319w0bwtde.jpg)

![](https://ws1.sinaimg.cn/large/006tKfTcgy1ft8egtg1naj31ji0fmgp2.jpg)

比照生成类代码中 addLogon(String var1) 方法体的内容以及展示的实例变量信息。可以看到生成类调用的 Callback 实例是 `CGLIB$CALLBACK_0` 
类型是 CglibAopProxy 的静态内部私有类 DynamicAdvisedInterceptor 。

![](https://ws1.sinaimg.cn/large/006tKfTcgy1ft8eu91oscj31gq0ayju5.jpg)

在 intercept(...) 上打个断点，跳过生成类 addLogon(String var1) 方法的下一个调用方法就是 intercept(...) 

**而且事实上，CGlib 生成类的内容基本也就是对调用方法进行的一种转发**

6. 继续 Debug ，对于生成类的内容，人肉了解并定位到触发的 Callback 实例或者 super.Xxx() 方法。

```plain
  __                    __                  
 / _| __ _ _ __   __ _ / _| ___ _ __   __ _ 
| |_ / _` | '_ \ / _` | |_ / _ \ '_ \ / _` |
|  _| (_| | | | | (_| |  _|  __/ | | | (_| |
|_|  \__,_|_| |_|\__, |_|  \___|_| |_|\__, |
                 |___/                |___/ 
```

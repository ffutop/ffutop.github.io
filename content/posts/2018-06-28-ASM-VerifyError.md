---
title: ASM-VerifyError错误信息解决
author: fangfeng
date: 2018-06-28
categories:
  - 技术
tag:
  - ASM
  - Java
  - CGlib
---

## 报错信息

```java
java.lang.VerifyError: class net.sf.cglib.core.DebuggingClassWriter overrides final method visit.(IILjava/lang/String;Ljava/lang/String;Ljava/lang/String;[Ljava/lang/String;)V
```

<!--more-->

## 背景

项目依赖的 
 - CGlib 版本是 2.2.2
 - ASM 版本是 3.3.1

## 问题定位

前两天刚粗略通读了 ASM ，结果就遇上这样一个问题。

从 `net.sf.cglib.core.DebuggingClassWriter` 看，这是 CGlib 的一个实现类

从描述 `overrides final method visit.(IILjava/lang/String;Ljava/lang/String;Ljava/lang/String;[Ljava/lang/String;)V` 
以及 DebuggingClassWriter 类的字节码反编译结果

```java
public class DebuggingClassWriter extends ClassWriter {
    public void visit(int version, int access, String name, String signature, String superName, String[] interfaces) {
        this.className = name.replace('/', '.');
        this.superName = superName.replace('/', '.');
        super.visit(version, access, name, signature, superName, interfaces);
    }
}
```

至少应该是 visit(...) 方法要重载的父类的被声明为 final 的 ClassWriter.visit(...) 才导致的问题。

但是从 ASM 3.3.1 版本的 ClassWriter 类可以看到

```java
    public void visit(int var1, int var2, String var3, String var4, String var5, String[] var6) {
        ...
    }
```

visit 方法并没有被声明为 final 。

但是，结合一定的背景知识，ASM 项目在加入了 OW2 组织后的新版本(asm-3.3.1 是未加入 OW2 前的最后一个版本)， 
ClassWriter 类的所有 visitXxx(...) 方法都被添加了 `final` 限制。

因此，怀疑是项目实际依赖了高版本的 asm (asm 4.x 及以上)

## 解决

经过确认，由于整个项目依赖关系复杂，有其它项目引入了 asm-4.1.jar。
而且由于使用 MAVEN 进行项目依赖管理，asm-4.1.jar 与依赖树根的深度更浅，
因此，最终打包的 war 里面确实引入了 asm-4.1.jar 。导致了这个问题。
```plain
  __                    __                  
 / _| __ _ _ __   __ _ / _| ___ _ __   __ _ 
| |_ / _` | '_ \ / _` | |_ / _ \ '_ \ / _` |
|  _| (_| | | | | (_| |  _|  __/ | | | (_| |
|_|  \__,_|_| |_|\__, |_|  \___|_| |_|\__, |
                 |___/                |___/ 
```

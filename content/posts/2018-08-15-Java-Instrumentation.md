---
title: Java Instrumentation
author: fangfeng
date: 2018-08-15
categories:
  - 技术
tags:
  - Java
  - ASM
  - BTrace
---

## Start

从现有的前置知识来说，我们能够认识到两个事实:

1. Java Class 通过 ClassLoader 进行加载。
   通过`全限定名`进行区分。当需要加载新的类时，ClassLoader 通过双亲委派机制判断是否已经加载过这个类。
   换句话说: Class 一经加载，就不会尝试重复加载 (至少按绝大多数人的认知来说，确实是的)
2. 有没有可能让被加载的 Class 与物理存储上的 .class 内容不同。
   当然也是完全可以做到的。不管怎么说，CGlib 和 Java Proxy 也是一个耳熟能详的概念吧
   (虽然可能不了解细节。在此，欢迎学习前置技能 [CGlib Enhancer 主流程源码解析](https://dormouse-none.github.io/2018-07-10-CGlib-Enhancer/) 和 [Java Proxy 源码解析](https://dormouse-none.github.io/2018-07-20-Java-Proxy/)。不过不影响本文后续内容)

另一个方面，也许绝大多数人都听说过所谓的`热部署`。但是究竟怎么才能做到 `热部署`(话题开得有点大哈。Y\_Y 本文不讲这个)

操作字节码一定是一个逃不开的话题，毕竟 Class 就是所谓的被加载到内存的字节码嘛。

如何操作字节码? ASM, CGlib, Java Proxy, Javassist ? 不过这些都要等到需要被操作的类被加载了才行啊，似乎有点晚...

Java 提供了一个可行的机制，用来在 ClassLoader 加载字节码之前完成对操作字节码的目的

## Instrumentation

`java.lang.instrument.Instrumentation` 类为提供直接操作 Java 字节码的又一个途径(虽然 Java Doc 的说明是用来检测 Java 代码的)

相信我这个说明是没有问题的。毕竟完成对代码检测的途径是直接修改字节码。

下列有两种方法可以达到目的

1. 当 JVM 以指示一个代理类的方式启动时，将传递给代理类的 premain 方法一个 Instrumentation 实例。
2. 当 JVM 提供某种机制在 JVM 启动之后某一时刻启动代理时，将传递给代理代码的 agentmain 方法一个 Instrumentation 实例。

话不多说，下面将全部以实例来展现对这种 JVM 检测机制(虽然例子已经脱离了*检测*的目的)的使用

<!--more-->

## 对各方法进行执行时间统计

### 随 JVM 一起启动
基本实例: 将对特定包 `me.fangfeng.client` 下的每个方法执行计时

首先了解一下 client 包的内容:

```java
package me.fangfeng.client;

/**
 * Main.java
 * 执行两个方法，rand() & sleep() 
 *
 * @author fangfeng
 * @since 2018/8/7
 */
public class Main {

    static void sleep() {
        try {
            Thread.sleep(5000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }

    public static void main(String[] args) {

        Rand rand = new Rand();

        for (int i=0;i<10;i++) {
            System.out.println(">>> start Rand.run() <<<");
            rand.run();
            System.out.println(">>> end Rand.run() <<<");

            System.out.println();

            System.out.println(">>> start Main.sleep() <<<");
            Main.sleep();
            System.out.println(">>> end MAin.sleep() <<<");

            System.out.println();
        }
    }
}
```

```java
package me.fangfeng.client;

/**
 * Rand.java
 * @author fangfeng
 * @since 2018/8/7
 */
public class Rand {

    public void run() {
        while (true) {
            double rand = Math.random();
            if (rand > 0.995) {
                System.out.println(String.format("get random, values %f", rand));
                return;
            }
        }
    }
}
```

接着，来构造一个代理类，以及最重要的 `premain` 方法

```java
package me.fangfeng.javaagent;

import java.lang.instrument.Instrumentation;

/**
 * Agent - 代理
 * 基于 JVM TI (JVM Tool Interface) 实现的 Java ClassFile 的增强
 * @author fangfeng
 * @since 2018/8/7
 */
public class Agent {

    // premain 将 JVM 初始化后，main(String... ) 执行前调用
    public static void premain(String args, Instrumentation instrumentation) {
        // new 一个转换器实例
        ClassTimer transformer = new ClassTimer();
        instrumentation.addTransformer(transformer);
    }

    // 之后的 agentmain(...) 将在这里提供，暂时隐去，避免对对读者产生干扰
}
```

```java
package me.fangfeng.javaagent;

import org.objectweb.asm.ClassReader;
import org.objectweb.asm.ClassWriter;
import org.objectweb.asm.Opcodes;

import java.lang.instrument.ClassFileTransformer;
import java.security.ProtectionDomain;

/**
 * @author fangfeng
 * @since 2018/8/7
 */
public class ClassTimer implements ClassFileTransformer {

    @Override
    public byte[] transform(ClassLoader loader, String className, Class<?> classBeingRedefined, ProtectionDomain protectionDomain, byte[] classfileBuffer) {
        // 这里涉及到了 ASM 的内容，主要目的是向每个方法块的开始及方法块的结束部分插入与计时器有关的代码
        // 如果想了解 ASM 的内容，请参阅 https://dormouse-none.github.io/2018-06-25-ASM-Core/  提供了一些基础性的内容，更多的请自行学习
        // 不了解具体内容将不影响对主体内容的理解
        ClassReader cr = new ClassReader(classfileBuffer);
        ClassWriter cw = new ClassWriter(ClassWriter.COMPUTE_FRAMES);
        MyClassWriter mcw = new MyClassWriter(Opcodes.ASM6, cw);
        cr.accept(mcw, ClassReader.EXPAND_FRAMES);
        return cw.toByteArray();
    }
}
```

其它代码略，详见附件。

Java 这种对操作字节码的支持有个坑爹的地方，就是不得不打包成 Jar 来使用。

具体来看一下

`me.fangfeng.javaagent` 包中包括 

![](https://ws4.sinaimg.cn/large/006tNbRwgy1fublsbbdw0j309o07g74u.jpg)

将被打包成 `agent.jar` 来使用

首先，来看一下需要打包在 `agent.jar` 的 **MANIFEST.MF** 的内容

```
Manifest-Version: 1.0
Class-Path: /Users/fangfeng/.m2/repository/org/ow2/asm/asm/6.1.1/asm-6.1.1.jar
Premain-Class: me.fangfeng.javaagent.Agent
Can-Retransform-Classes: true
```

再来个 SHELL 脚本，用来给打包这个 Jar

```sh
#!/bin/bash

# 编译 me.fangfeng.javaagent 包下的类
javac -cp .:/Users/fangfeng/.m2/repository/org/ow2/asm/asm/6.1.1/asm-6.1.1.jar me/fangfeng/javaagent/Agent.java me/fangfeng/javaagent/ClassTimer.java me/fangfeng/javaagent/MyClassWriter.java me/fangfeng/javaagent/MyMethodWriter.java me/fangfeng/javaagent/StaticTimer.java

# 打包 me.fangfeng.javaagent 的 .class -> agent.jar
jar cvfm agent.jar MANIFEST-agent.MF me/fangfeng/javaagent/Agent.class me/fangfeng/javaagent/ClassTimer.class me/fangfeng/javaagent/MyClassWriter.class me/fangfeng/javaagent/MyMethodWriter.class me/fangfeng/javaagent/StaticTimer.class

# 编译 me.fangfeng.client 包下的类
javac me/fangfeng/client/Main.java me/fangfeng/client/Rand.java

# 以 me.fangfeng.client.Main 作为主类启动
java -javaagent:agent.jar me.fangfeng.client.Main
```

执行后，可以看到类似如下内容:

![](https://ws2.sinaimg.cn/large/006tNbRwgy1fubmp354ejj30tq0rmtdp.jpg)

而直接用 `java me.fangfeng.client.Main` 的执行结果是:

![](https://ws4.sinaimg.cn/large/006tNbRwgy1fubmqmj4uhj30j00f0dho.jpg)

从理论上来讲，`-javaagent:agent.jar` 配合 `agent.jar` 中的 MANIFEST.MF 文件，
使得 JVM 在初始化之后触发了被声明为 `Pre-Main` 的 me.fangfeng.javaagent.Agent 类的 premain(...) 方法。

并为 ClassLoader 在加载类的流程上增加了一层**拦截器** (这里是 ClassTimer.java 类，它实现了 `ClassFileTransformer` 接口

另外，`Can-Retransform-Classes: true` 的配置使得 ClassTimer 被允许对字节码进行重新转换。(而操作字节码是通过 ASM 来实现的)
 
### 在运行中进行增强

随着程序启动时直接使用了 `-javaagent` 选项。

那么是否存在在程序运行中进行额外代理操作的支持呢？当然是可以的。这里要借助 Java 提供的另一个类 com.sun.tools.attach.VirtualMachine 。

启动一个新的进程来连接到 正在运行中的进程，并令其加载 java agent。

基本的类与上一节的描述相同，主要是包 `me.fangfeng.javaagent.*` 和 `me.fangfeng.client.*`

新增一个类 `me.fangfeng.javaagent.Main` 用来启动另一个进程，并要求运行中的 java 进程加载 agent.jar 来进行增强。

```java
package me.fangfeng.javaagent;

import com.sun.tools.attach.AgentInitializationException;
import com.sun.tools.attach.AgentLoadException;
import com.sun.tools.attach.AttachNotSupportedException;
import com.sun.tools.attach.VirtualMachine;

import java.io.IOException;

/**
 * @author fangfeng
 * @since 2018/8/7
 */
public class Main {

    public static void main(String[] args) throws IOException, AttachNotSupportedException, AgentLoadException, AgentInitializationException {
        VirtualMachine vm = null;
        try {
            // 通过 VirtualMachine 连接到 运行中的进程 (可以通过 jps 找到进程号)
            vm = VirtualMachine.attach(<PID>);
            vm.loadAgent(<agent.jar 的路径>);
        } finally {
            if (vm != null) {
                vm.detach();
            }
        }
    }
}
```

```java
public class Agent {

    public static void premain(String args, Instrumentation instrumentation) {
        ClassTimer transformer = new ClassTimer();
        instrumentation.addTransformer(transformer);
    }

    // 现在在 Agent.java 上补上 agentmain(...) 的具体实现
    public static void agentmain(String args, Instrumentation instrumentation) throws UnmodifiableClassException {
        System.out.println("SUCCESS AGENTMAIN");
        ClassTimer transformer = new ClassTimer();
        // add Transformer
        instrumentation.addTransformer(transformer, true);
        // 对 Rand.class 进行重新转换
        instrumentation.retransformClasses(Rand.class);
    }
}
```

其它内容基本相同

首先需要先打包 agent.jar 。当然，如果是顺着本文的顺序进行本机实验，则 agent.jar 已经存在

先启动进程 `java me.fangfeng.client.Main`

通过 `jps` 获取 Main 进程的 **PID**

在 `java me.fangfeng.javaagent.Main` 中替换上进程号，并执行

![](https://ws2.sinaimg.cn/large/006tNbRwgy1fuboclcelsj30tu152dlw.jpg)

从执行结果可以看到，原进程首先正常执行代码，等到被 load Agent 之后，字节码已经有了新的变化，从而导致输出结果动态的产生了变化。

*当然，需要注意的是，执行中的进程被要求 load Agent 之后，运行中的 Class 将被改写，并始终如此，知道进程终止。再下一次重新启动*

## BTrace

以上描述的内容也可以理解为是 BTrace 实现的基础。毕竟，JVMTI (JVM Tool Interface) 原本的目的就是赋予使用者一个在运行中
查询系统各项数据的权利

当然，实现上，上述代码直接将各种增强(计时)硬编码到该进程中，同时统一使用了该进程的输入输出。

但是，BTrace 通过 Socket 将这些分离，检测代码通过 Socket 发回新的进程来维持输入输出。

在此，不再细说。

## 附录

\[1\]. 示例代码: [instru.zip](https://github.com/DorMOUSE-None/Repo/raw/master/instru.zip) 
\[2\]. [java.lang.instrument.Instrumentation](https://docs.oracle.com/javase/8/docs/api/java/lang/instrument/Instrumentation.html)
\[3\]. [Package java.lang.instrument](https://docs.oracle.com/javase/8/docs/api/java/lang/instrument/package-summary.html)

```plain
  __                    __                  
 / _| __ _ _ __   __ _ / _| ___ _ __   __ _ 
| |_ / _` | '_ \ / _` | |_ / _ \ '_ \ / _` |
|  _| (_| | | | | (_| |  _|  __/ | | | (_| |
|_|  \__,_|_| |_|\__, |_|  \___|_| |_|\__, |
                 |___/                |___/ 
```

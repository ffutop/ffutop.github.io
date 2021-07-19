---
title: 【Java】API 参数误定义的后果
author: fangfeng
date: 2019-02-27
categories:
  - 技术
tags:
  - Java
  - API
---

工程项目中，定义 API 总是一个慎之又慎的操作。不能少，不满足调用方的需求就惨了；也不能多，不然就乱套了，自己维护困难，调用方也开始了自我发挥。虽然足够慎重，但绝大多数都逃不过最终不得不“改” API 的情况。今天要讨论的是在同一个类内同方法名不同参数（入参/出参）的情况。

想要做到同方法名不同入参，很简单，就是“重载（Overload）”，日常都在使用。不再赘述！

想要做到同方法名不同出参，答案就不再那么肯定了。当然，如果问把`void add(int)` API 改写成 `int add(int)`，可能得到的大多数回答都是可以。

<!--more-->

## 看山是山

首先举一个具体点的例子来描述（为了方便，就不定义`CountService`的接口类了）

```java
package com.ffutop.signature;

/**
 * 主类
 * @author fangfeng
 * @since 2019-02-26
 */
public class Main {

    public static void main(String[] args) {
        CountService countService = new CountService();
        countService.add(1);
        System.out.println(String.format("currentValue = %d", countService.getCurrentValue()));
    }
}
```

```java
package com.ffutop.signature;

/**
 * @author fangfeng
 * @since 2019-02-26
 */
public class CountService {

    private int currentValue = 0;

    /**
     * 请把 add(int) 理解成 API
     * 虽然已经做了实现
     */
    public void add(int addend) {
        currentValue += addend;
    }

    public int getCurrentValue() {
        return currentValue;
    }
}
```

现在已经有 `void add(int)` 方法，完成的工作是累加。现在要把 API 改成 `int add(int)`，是否可以呢？看似本来没有返回值，现在只是加上一个返回值，对于原来的代码没有任何问题，而新代码又能拿到int类型的返回值，皆大欢喜啊。

## 看山不是山

先给大家执行下代码吧！（请严格按照顺序，且避免使用IDE，否则将得不到预想的结果）

```shell
$ # 准备好两个类的代码（CountService的API是 `void add(int)`）
$ # 编译Main类
$ javac com/ffutop/signature/Main.java
$  
$ # 修改CountService的API为`int add(int)`
$ # 编译 CountService 类
$ javac com/ffutop/signature/CountService.java
$ 
$ # 执行主程序
$ java com.ffutop.signature.Main
Exception in thread "main" java.lang.NoSuchMethodError: com.ffutop.signature.CountService.add(I)V
	at com.ffutop.signature.Main.main(Main.java:11)
```

很遗憾，执行失败了，报了个Error（没有这样的方法）。方法写的是 `com.ffutop.signature.CountService.add(I)V` 。简单的翻译一下就是需要`类名+方法名=x.y.CountService.add`，且入参为int，出参为void的方法（想了解更多请优先学习[Java ClassFile Format](https://docs.oracle.com/javase/specs/jvms/se9/html/jvms-4.html#jvms-4.4)）。

那么，现在得到的结论是不行。

## 看山还是山

那么，就一定不行吗？都是同名方法，不同参数。凭什么改个入参是可以的，改个出参就不行了。

从 Java 的角度来说，一定不行。“同一类中不能存在两个名字及描述符完全相同的方法”

但是，如果从 JVM 的角度来说，完全是可行的。“在class文件中，两个方法可以拥有完全相同的特征签名，前提是返回值不能相同”

什么意思呢？我们都知道建立在 JVM 之上的语言不只有 Java。像 Groovy、Kotlin 等语言都实现了各自的区别于 Java 独有的特征。这些特征的实现都是依赖于 JVM，但为什么 Java 没有呢？只能说 Java 语言的规范是 JVM 的子集（好像这话有点不严谨啊）

通过直接操作字节码，就能够达到在同一个类中建立两个同名、相同入参，但返回值不同的方法。
先通过`javap`命令看看最终提供的`CountService.class`

```shell
$ javap com.ffutop.signature.CountService
public class com.ffutop.signature.CountService {
  public int add(int);
  public com.ffutop.signature.CountService();
  public void add(int);
  public int getCurrentValue();
}
```

有两个同名的方法`add(int)`，至于执行，也会相当顺利。

还是写个程序来说明，在原有 `Main.java` 的基础上，再创建一个全限定名为 `com.ffutop.signature.other.Main2` 的类

```java
package com.ffutop.signature.other;
import com.ffutop.signature.CountService;

/**
 * @author fangfeng
 * @since 2019-02-27
 */
public class Main2 {

    public static void main(String[] args) {
        CountService countService = new CountService();
        System.out.println(String.format("currentValue = %d", countService.add(1)));
    }
}
```

与 Main.java 比较，很明显的就是一个调用了 `CountService` 的 `int add(int)` 方法，而另一个调用 `void add(int)` 方法。

那么是否有效呢？先来验证一下（这里要模拟的一个场景是，CountService 类(只有 `void add(int)` 方法)作为二方库C.jar version 1.0 的主要服务类。CountService 类(同时有 `void add(int)` 和 `int add(int)` 方法)作为二方库C.jar version 2.0 的主要服务类。Main 类根据 C.jar version 1.0 做的编译，而 Main2 类根据 C.jar version 2.0 做的编译。在 Main 和 Main2 类运行时只提供 C.jar version 2.0）

```shell
$ # 编译 Main 类和 CountService 类
$ javac com/ffutop/signature/Main.java
$
$ # 操作 CountService.class 字节码，增加方法 `int add(int)` 
$ # 通过 ASM 实现，详见附录源码 com/ffutop/signature/support/Generator.java
$ 
$ # 编译 Main2 类 (提供 classpath，即根据 CountService.class 编译)
$ javac com/ffutop/signature/other/Main2.java -classpath .
$
$ # 验证
$ java com.ffutop.signature.Main
currentValue = 1
$ java com.ffutop.signature.other.Main2
currentValue = 1
$ # OK，验证通过
```

## 总结

一旦 API 定义错误，并已经作为二方库提供给调用方。后果必然是灾难性的。至少，只能重新定义 API 。如果对于入参定义错误，Java 语言级别也能够支持。但如果是返回值，如果非要保持方法名一致，那就不得不下沉到 JVM 级别来进行处理了。当然，在工程项目中是否应该直接操作字节码？至少个人还没直接接触过。

做个记录，未来可以翻一翻，至少是一种可行的解决方案。

Update: 如果直接操作字节码为添加了不同返回值的同名同参数方法，可能引起调用方静态编译失败。此操作需慎之又慎。也许提供类加载器加载时的字节码操作能更加完美地解决这个问题。

## 参考

[源码.zip](https://raw.githubusercontent.com/DorMOUSE-None/Repo/master/signature.zip)
```plain
  __                    __                  
 / _| __ _ _ __   __ _ / _| ___ _ __   __ _ 
| |_ / _` | '_ \ / _` | |_ / _ \ '_ \ / _` |
|  _| (_| | | | | (_| |  _|  __/ | | | (_| |
|_|  \__,_|_| |_|\__, |_|  \___|_| |_|\__, |
                 |___/                |___/ 
```

---
title: Java 安全访问与权限控制
author: fangfeng 
date: 2018-07-04
categories:
  - 技术
tags:
  - Java
  - Security
  - Permission
---

## 绪论

*本文只是对 Java 安全访问与权限控制的基础性探究。*

**本节与全文内容无关，如无兴趣阅读，可以跳过**

了解 Java 安全访问相关内容的初衷，是准备在项目中利用 Java 标准库提供的 ServiceLoader 对 SPI 实现类进行"自动发现"和加载。
这对于将本项目作为二方库来依赖的上层项目将更为方便，只需要
1. 在 `META-INF.services` 目录下配置被命名为 SPI 接口全限定名的文件及添加相关内容
2. 由项目的注册管理器触发下列 Java 代码

```java
{
    ServiceLoader<XxxPolicy> xxxPolicyServiceLoader = ServiceLoader.load(XxxPolicy.class);
    for (Iterator<XxxPolicy> it = xxxPolicyServiceLoader.iterator(); it.hasNext(); ) {
        XxxPolicy xxxPolicy = it.next();
        // ... more code ...
    }
}
```
就可以完成一个新的 SPI 策略的注册工作。

但是，在尝试实现，了解了 ServiceLoader 源码，以及 DriverManager 和 mysql-connection-java-<version>.jar 在注册 Driver 相关的代码。
发现怎么也绕不开 Java 安全访问相关的内容。诸如下列这段来自 DriverManager.loadInitialDrivers() 的代码:

```java
AccessController.doPrivileged(new PrivilegedAction<Void>() {
    public Void run() {

        ServiceLoader<Driver> loadedDrivers = ServiceLoader.load(Driver.class);
        Iterator<Driver> driversIterator = loadedDrivers.iterator();

        try{
            while(driversIterator.hasNext()) {
            driversIterator.next();
            }
        } catch(Throwable t) {
                // Do nothing
        }
        return null;
    }
});
```

诸如 AccessController, Permission, SecurityManager 的代码始终是一个绕不开的主旋律。

为了探究这部分控制对项目中 ServiceLoader 的真正作用以及其编码意义，开始了对本文所描述的主体内容的初步了解。

<!--more-->

## 从现象开始...

在通过 `java` 命令执行本地代码时，偶尔/经常会出现文件I/O操作。

```java
public static void main(String... args) throws IOException {

    System.out.println(System.getSecurityManager());

    FileInputStream fis = new FileInputStream("/Users/fangfeng/test.in");
    for (int chr; (chr = fis.read()) != -1;) {
        System.out.print((char) chr);
    }
    fis.close();
}
```

诸如上面这段代码，意在读取外部路径下 `test.in` 文件(不要放在项目路径下，文本内容为 `0123456789`)。当然，还包括打印 System.getSecurityManager().toString() 。

正常情况下，这都是能够执行成功，结果为:

```plain
null
0123456789
```

但是，通过在命令行 `java` 中添加选项 `-Djava.security.manager`，再次执行代码，结果为:

```plain
java.lang.SecurityManager@4e25154f
Exception in thread "main" java.security.AccessControlException: access denied ("java.io.FilePermission" "/Users/fangfeng/test.in" "read")
	at java.security.AccessControlContext.checkPermission(AccessControlContext.java:472)
	at java.security.AccessController.checkPermission(AccessController.java:884)
	at java.lang.SecurityManager.checkPermission(SecurityManager.java:549)
	at java.lang.SecurityManager.checkRead(SecurityManager.java:888)
	at java.io.FileInputStream.<init>(FileInputStream.java:127)
	at java.io.FileInputStream.<init>(FileInputStream.java:93)
	at me.fangfeng.security.SecurityTest.main(SecurityTest.java:16)
```

现在已经能够获取到 `System.getSecuriryManager` 的实例。
但是想要读取 `test.in` 文件却失败了，表现为 access denied（访问被拒绝）。

现在，在用户目录下(这里是 /Users/fangfeng, 不同系统不同用户请做相应修改) 添加 `.java.policy` 文件，添加下列文本:

```plain
grant {
    permission java.io.FilePermission "/Users/fangfeng/test.in", "read";
};
```

再次 `java -Djava.security.manager <class's path>`，不仅能够得到 SecurityManager 的实例，同时也读取到了文本内容。

```plain
java.lang.SecurityManager@3af49f1c
0123456789
```

<hr/>

到此为止，应该已经能够感受到 Java 对安全访问的控制在文件I/O上的体现了。

## 安全控制下的操作

**在开始下列内容之前，需要提前了解一个前提:** 
**Java 对操作权限的控制是通过检查当前线程操作上下文的代码是否存在要求的操作权限实现的(除了特权的声明以及其它还需要控制别的线程上下文权限的情况)**

上一节演示了权限控制下本地代码对本地资源的访问。但是，安全控制真的有必要吗？
- 就本地程序而言，不必要。事实上，Java 也确实是这么做的。平时在本地利用 `java` 命令执行程序，都不会受到任何限制。Java 启动时默认不会装载 SecurityManager，也就不会触发对各种操作的权限验证。
- 可如果程序运行与网络上其它程序交互，甚至直接加载来自网络的字节码。那么，权限控制确实是有必要的。至少，如果不存在权限控制，来自网络的恶意代码访问整台机器的任意资源都将畅通无阻。

### SecurityManager

SecurityManager 是整个访问控制的管理器和基本入口。所有涉及到权限控制的代码，都会类似地存在下列这般的代码:

```java
SecurityManager security = System.getSecurityManager();
// 如果系统存在安全管理器
if (security != null) {
    // 调用 SecurityManager 中以 check 开头的方法
    security.checkXxx(...);
}
```

`security.checkXxx(...)` 是没有返回值的(除了 checkTopLevelWindow)；对于没有操作权限的代码，直接抛出异常

至于到底在 check 什么，Java 中定义了包括文件(File)、套接字(Socket)、网络(Net)、安全性(Security)、运行时(Runtime)、属性(Property)、AWT、反射(Reflect)和可序列化(Serializable) 九类权限。

通常，security.checkXxx(...) 方法将构造一个 XxxPermission(...) 对象来调用 SecurityManager 提供的统一方法 checkPermission(Permission) 。

```java
// 以 checkRead(name) 为例
public void checkRead(String file) {
    checkPermission(new FilePermission(file, SecurityConstants.FILE_READ_ACTION));
}

// 调用 checkPermission(Permission) 方法
public void checkPermission(Permission perm) {
    // 直接调用 访问控制器 来对权限进行鉴别
    java.security.AccessController.checkPermission(perm);
}
```

### AccessController

AccessController 用于与访问控制相关的操作和决定。

> AccessController 类用于以下三个目的：
> 
> 基于当前生效的安全策略决定是允许还是拒绝对关键系统资源的访问
> 将代码标记为享有“特权”，从而影响后续访问决定，以及
> 获取当前调用上下文的“快照”，这样便可以相对于已保存的上下文作出其他上下文的访问控制决定

### 小结

总的来说，Java 的访问控制就是通过针对不同的操作构建不同的 Permission 对象，然后通过与 AccessControllerContent 持有的各代码的权限进行比对，
从而判断代码是否存在相应的访问权限。

*所谓的Permission判断将是对当前线程的调用栈的每个调用者逐一倒序递归，判断是否拥有权限。如果其中的某个代码被声明为 Privileged，则直接认为是拥有权限*

**更多的关于 checkPermission(...) 调用相关的内容，可以自行查阅资料进行学习，在此不做过多陈述。:)**

## 为操作赋权

上一节讲完了如果对当前项目配置了 SecurityManager ，将对各种敏感操作进行访问控制，并且将根据整个调用链被赋予的权限来决定是允许执行还是抛出 access denied。

但是，究竟怎么才能够给 code 赋予权限呢？

回顾前一节的内容，在基本探究中，其实一个能够看到 
```java
grant {
    permission java.io.FilePermission "/Users/fangfeng/test.in", "read";
};
```
这就是一种赋权的操作。

通过在指定的文件中写入符合特定规则的代码，来完成对 code 的赋权。[Default Policy Implementation and Policy File Syntax](https://docs.oracle.com/javase/8/docs/technotes/guides/security/PolicyFiles.html)

在项目启动的时候，默认就会读取 $JAVA\_HOME/jre/lib/security/java.policy 以及 ${user.home}/.java.policy 两个文件的赋权内容，并做缓存给后面代码使用。

当然，Java 也提供指定自定义的赋权文件，通过 -Djava.security.policy=<policy file> 或者 -Djava.security.policy==<policy file> 。

## 参考

\[1\]. Java Document - Security. https://docs.oracle.com/javase/8/docs/technotes/guides/security/index.html

```plain
  __                    __                  
 / _| __ _ _ __   __ _ / _| ___ _ __   __ _ 
| |_ / _` | '_ \ / _` | |_ / _ \ '_ \ / _` |
|  _| (_| | | | | (_| |  _|  __/ | | | (_| |
|_|  \__,_|_| |_|\__, |_|  \___|_| |_|\__, |
                 |___/                |___/ 
```

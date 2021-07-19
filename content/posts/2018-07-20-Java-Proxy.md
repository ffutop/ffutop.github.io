---
title: Java Proxy 源码解析
author: fangfeng
date: 2018-07-20
categories:
  - 技术
tag:
  - Java
  - Proxy
---

在 Java 整个生态里面, 通用的有两类动态代理的应用: Java Proxy 与 CGlib 代理。

从宽泛的区别来说，Java Proxy 只能对接口进行增强，而 CGlib 同时适用于类和接口的增强。
而且，业内普遍的认知是，CGlib 动态代理较之于 Java Proxy 在生成字节码的速度上也更为高效。

<!--more-->

## 从实例开始...

下面，首先来了解一下 Java Proxy 的使用编码:

ICodeFactory 接口, 作为将被 Java 动态代理增强的基本接口
```java
public interface ICodeFactory {

    Code getCode();

    void setCode(Code code);

}
```

CodeFactory 类, 作为 ICodeFactory 接口的实现类。被视为是真正被动态代理增强的内容，:) 因为接口方法并不存在方法体(当然，Java 8 及以上的 default 请容我忽视)
```java
public class CodeFactory implements ICodeFactory {

    private Code code;

    public Code getCode() {
        return new Code();
    }

    @Override
    public void setCode(Code code) {
        this.code = code;
    }
}
```

辅助类 Code 的内容(简单写，请忽略不符合实际编程规范的一些内容):
```java
public class Code {

    public int codeA;

    public String codeB;

}
```

构建增强的代码逻辑:
```java
// main(String[]) 方法
public static void main(String[] args) throws Exception {

    // 内部匿名类，用于动态代理生成类的构造方法及其它内容，后面讲。
    InvocationHandler handler = new InvocationHandler() {

        Object obj = new CodeFactory();

        @Override
        public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {

            if (method.getName() == "getCode") {
                Code code = (Code) method.invoke(obj, args);
                code.codeA = code.codeA + 100;
                code.codeB = "Proxied: " + code.codeB;
                return code;
            } else {
                return method.invoke(obj, args);
            }
        }
    };

    // 构建生成类实例
    Class<?> clazz = Proxy.getProxyClass(ICodeFactory.class.getClassLoader(), ICodeFactory.class);
    // 获取生成类对象
    ICodeFactory factory = (ICodeFactory) clazz.getConstructor(new Class[] { InvocationHandler.class }).newInstance(new Object[] { handler });

    // 调用 getCode() 方法
    Code code = factory.getCode();
    // 打印参数
    System.out.println(code.codeA);
    System.out.println(code.codeB);
}
```

从默认预期来讲，我们是对接口进行增强。接口方法不存在方法体，也许将在 `factory.getCode()` 时执行失败？
或者结合已有的认知，成功调用 `CodeFactory.getCode()` 方法，并获得 `new Code()` 。

那么，`code.codeA` `code.codeB` 的具体值将是什么？

执行结果将是:

```plain
100
Proxied: null
```

## 了解 Proxy 的内容逻辑

从 `Proxy.getProxyClass(ClassLoader, Class<?>...)` 入手，下面将展开对 `Proxy` 具体执行逻辑的探究。

```java
public static Class<?> getProxyClass(ClassLoader loader, Class<?>... interfaces) throws IllegalArgumentException {
    // 对 interfaces 数组的浅拷贝
    final Class<?>[] intfs = interfaces.clone();
    final SecurityManager sm = System.getSecurityManager();
    if (sm != null) {
        // 如果配置了安全管理器，那么需要确认 Proxy 有创建新的代理类的许可
        checkProxyAccess(Reflection.getCallerClass(), loader, intfs);
    }

    // 调用 
    return getProxyClass0(loader, intfs);
}
```

```java
private static Class<?> getProxyClass0(ClassLoader loader, Class<?>... interfaces) {
    // 确认需要进行增强的接口数量少于等于 65535，详细原因请去了解 ClassFile 规范(规范中的接口上限)
    if (interfaces.length > 65535) {
        throw new IllegalArgumentException("interface limit exceeded");
    }

    // 如果需要由指定 ClassLoader 加载, 实现给定 interfaces 的代理类已经存在, 
    // 将直接返回已经缓存过的拷贝
    // 否则，通过 ProxyClassFactory 创建新的代理类
    return proxyClassCache.get(loader, interfaces);
}
```

现在需要额外来考察一下 proxyClassCache 的具体类型。

```java
private static final WeakCache<ClassLoader, Class<?>[], Class<?>>
        proxyClassCache = new WeakCache<>(new KeyFactory(), new ProxyClassFactory());
```

`WeakCache`, 从名称来将，是一个缓存，同时应该使用了 WeakReference 技术

进入 `WeakCache.get(...)` 方法体

```java
public V get(K key, P parameter) {
    // parameter 传入的是接口数组，要求不能为空
    Objects.requireNonNull(parameter);

    // 删除过期元素
    expungeStaleEntries();

    // 构建一个 WeakReference 对象(key 表示 ClassLoader)
    Object cacheKey = CacheKey.valueOf(key, refQueue);

    // lazily install the 2nd level valuesMap for the particular cacheKey
    ConcurrentMap<Object, Supplier<V>> valuesMap = map.get(cacheKey);
    if (valuesMap == null) {
        ConcurrentMap<Object, Supplier<V>> oldValuesMap
                = map.putIfAbsent(cacheKey,
                valuesMap = new ConcurrentHashMap<>());
        if (oldValuesMap != null) {
            valuesMap = oldValuesMap;
        }
    }

    // 创建次级 Key
    // 同时检索是否存在过去有同一个 ClassLoader 加载的表示 Supplier<V>
    Object subKey = Objects.requireNonNull(subKeyFactory.apply(key, parameter));
    Supplier<V> supplier = valuesMap.get(subKey);
    Factory factory = null;

    while (true) {
        if (supplier != null) {
            // supplier might be a Factory or a CacheValue<V> instance
            // 最终 supplier 将非空，同时调用 .get() 方法获取 Class 类
            V value = supplier.get();
            if (value != null) {
                return value;
            }
        }
        // else no supplier in cache
        // or a supplier that returned null (could be a cleared CacheValue
        // or a Factory that wasn't successful in installing the CacheValue)
        // 未找到过去加载的记录

        // 懒加载一个 Factory 
        if (factory == null) {
            factory = new Factory(key, parameter, subKey, valuesMap);
        }

        if (supplier == null) {
            supplier = valuesMap.putIfAbsent(subKey, factory);
            if (supplier == null) {
                // successfully installed Factory
                supplier = factory;
            }
            // else retry with winning supplier
        } else {
            if (valuesMap.replace(subKey, supplier, factory)) {
                // successfully replaced
                // cleared CacheEntry / unsuccessful Factory
                // with our Factory
                supplier = factory;
            } else {
                // retry with current supplier
                supplier = valuesMap.get(subKey);
            }
        }
    }
}
```

```java
@Override
public synchronized V get() { // serialize access
    // re-check
    Supplier<V> supplier = valuesMap.get(subKey);
    if (supplier != this) {
        // something changed while we were waiting:
        // might be that we were replaced by a CacheValue
        // or were removed because of failure ->
        // return null to signal WeakCache.get() to retry
        // the loop
        return null;
    }
    // else still us (supplier == this)

    // create new value
    V value = null;
    try {
        // 触发 valueFactory.apply() 真正的构建
        value = Objects.requireNonNull(valueFactory.apply(key, parameter));
    } finally {
        if (value == null) { // remove us on failure
            valuesMap.remove(subKey, this);
        }
    }
    // the only path to reach here is with non-null value
    assert value != null;

    // wrap value with CacheValue (WeakReference)
    CacheValue<V> cacheValue = new CacheValue<>(value);

    // put into reverseMap
    reverseMap.put(cacheValue, Boolean.TRUE);

    // try replacing us with CacheValue (this should always succeed)
    if (!valuesMap.replace(subKey, this, cacheValue)) {
        throw new AssertionError("Should not reach here");
    }

    // successfully replaced us with new CacheValue -> return the value
    // wrapped by it
    return value;
}
```

上面两段代码的内容，都是在反复确认将要构建的生成类是否在过去曾经创建过，如果是，则直接返回过去构建的类实例;
否则才会尝试创建，并最终将这个构建的类也进行缓存。

下面这段代码来自于 `Proxy` 的内部类 `ProxyClassFactory` 
这部分，也终于开始了对代理类字节码的统筹性构造的内容。

```java
public Class<?> apply(ClassLoader loader, Class<?>[] interfaces) {

    Map<Class<?>, Boolean> interfaceSet = new IdentityHashMap<>(interfaces.length);
    for (Class<?> intf : interfaces) {
        /*
         * 校验当前接口是由同一个 ClassLoader (将要用来加载新的代理类)加载的
         */
        Class<?> interfaceClass = null;
        try {
            interfaceClass = Class.forName(intf.getName(), false, loader);
        } catch (ClassNotFoundException e) {
        }
        if (interfaceClass != intf) {
            throw new IllegalArgumentException(
                    intf + " is not visible from class loader");
        }
        /*
         * 校验当前 Class 对象确实是一个接口
         */
        if (!interfaceClass.isInterface()) {
            throw new IllegalArgumentException(
                    interfaceClass.getName() + " is not an interface");
        }
        /*
         * 校验当前接口没有被要求重复进行代理增强
         */
        if (interfaceSet.put(interfaceClass, Boolean.TRUE) != null) {
            throw new IllegalArgumentException(
                    "repeated interface: " + interfaceClass.getName());
        }
    }

    String proxyPkg = null;     // 生成的代理类所属的 package
    int accessFlags = Modifier.PUBLIC | Modifier.FINAL;

    /*
     * 如果代理的接口是非 PUBLIC ，那么生成的代理类的接口将与这个非 PUBLIC 接口
     * 在同一个包下。
     * 如果代理的接口有多个非 PUBLIC ，且分别位于不同的包下，那么抛出异常
     */
    for (Class<?> intf : interfaces) {
        int flags = intf.getModifiers();
        if (!Modifier.isPublic(flags)) {
            accessFlags = Modifier.FINAL;
            String name = intf.getName();
            int n = name.lastIndexOf('.');
            String pkg = ((n == -1) ? "" : name.substring(0, n + 1));
            if (proxyPkg == null) {
                proxyPkg = pkg;
            } else if (!pkg.equals(proxyPkg)) {
                throw new IllegalArgumentException(
                        "non-public interfaces from different packages");
            }
        }
    }

    // 如果代理的接口全是 PUBLIC 的，那么使用默认的包 com.sun.proxy
    if (proxyPkg == null) {
        // if no non-public proxy interfaces, use com.sun.proxy package
        proxyPkg = ReflectUtil.PROXY_PACKAGE + ".";
    }

    /*
     * 为将要生成的代理类选择一个全限定名
     * 规则是 包名 + "$Proxy" + <唯一递增的id, 从0开始编号>
     */
    long num = nextUniqueNumber.getAndIncrement();
    String proxyName = proxyPkg + proxyClassNamePrefix + num;

    /*
     * 生成一个特殊的代理类的字节码
     * byte[] proxyClassFile 就等同于 .class 文件的全部二进制内容。
     * 与 .class 文件的区别就在于一个写在了外存上，另一个完全在内存中创建
     */
    byte[] proxyClassFile = ProxyGenerator.generateProxyClass(
            proxyName, interfaces, accessFlags);
    try {
        /**
          * 调用 ClassLoader 的 defineClass0(...) 方法加载字节码内容，构造 Class 对象
          * 本来 defineClass0(...) 方法是定义在 ClassLoader 中，但这里额外声明了一个本地方法 defineClass0(...)
          * 想来实现也是类似的，最终的目的也是加载 Class 
          */
        return defineClass0(loader, proxyName,
                proxyClassFile, 0, proxyClassFile.length);
    } catch (ClassFormatError e) {
        /*
         * A ClassFormatError here means that (barring bugs in the
         * proxy class generation code) there was some other
         * invalid aspect of the arguments supplied to the proxy
         * class creation (such as virtual machine limitations
         * exceeded).
         */
        throw new IllegalArgumentException(e.toString());
    }
}
```

在完成基础性的校验，并构造了生成类的类名等内容后，
`ProxyGenerator.generateProxyClass` 将开始构造 [ClassFile](https://docs.oracle.com/javase/specs/jvms/se9/html/jvms-4.html) 的具体内容。

关于这部分生成字节码的内容，请在了解了 ClassFile 的基础性内容之后，在自行学习，内容全部在 ProxyGenerator 类中。

这部分的规则是(在构建新的代理生成类时):

- 额外添加三个 Object 的方法 (`hashCode`, `equals`, `toString`)
- 逐一读取所有需要增强的接口的方法，并对重复方法进行判断; 如果方法名相同，入参数量与类型相同，但返回参数不同，则抛出异常
- 在遍历所有接口的方法的时候，同时也将为新的生成类构造新的字段(内容就是通过反射获取该方法)
- 最后将新的生成类将要包含的所有字段与方法进行构造，同时维护一个常量池内容
- 输出这些内容的二进制表示 byte[];

## 如何对方法增强

想必上了上一节，虽然对 Java 动态代理的调用链有了一定的了解。

但是，究竟 Proxy 是如何完成对实现类方法的增强呢？

也许我们需要先看一下生成类 .class 文件的一些内容来对这个命题形成基础性的影响。

```java
public final Code getCode() throws  {
    try {
        return (Code)super.h.invoke(this, m4, (Object[])null);
    } catch (RuntimeException | Error var2) {
        throw var2;
    } catch (Throwable var3) {
        throw new UndeclaredThrowableException(var3);
    }
}
```

可以看到，生成类的 `getCode()` 方法几乎没有什么实质性的内容, 只是 `super.h.invoke(...)` 。

`h` 实例变量是什么？`InvocationHandler` 的一个实例对象。

事实上，Java Proxy 生成类的每个方法结构几乎都是类似的。方法体的内容就是通过实例持有的 `h` 变量分发实际的操作指令

至于 h 的具体内容。:) 有编程者自定义，并在 Proxy 生成字节码，并被 ClassLoader 加载后，在准备实例化的时候作为构造参数进行传入。

```java
ICodeFactory factory = (ICodeFactory) clazz.getConstructor(new Class[] { InvocationHandler.class }).newInstance(new Object[] { handler });
```

至于具体将做哪些增强，调用例如上例的 `ICodeFactory` 的那个实现类的方法，全部有使用者自定义。

## 生成类的反编译结果

仍然以本文开始的 ICodeFactory 的增强为例，看一下究竟通过 Java Proxy 动态代理生成的字节码的反编译结果究竟是什么？
并以此来对这种动态代理机制形成更为直观的印象

在 Java 源代码中，虽然默认不会输出生成类的二进制内容，但是，仍然预留了打印 .class 文件的可选项。

有兴趣的同学可以看一下 `ProxyGenerator.saveGeneratedFiles` 字段的内容，这就决定是在构造代理类后是否存储到外存中。

想要看到代理类的字节码，直接通过在 Java 执行程序中添加一行代码

```java
System.setProperty("sun.misc.ProxyGenerator.saveGeneratedFiles", "true");
```

或者在启动程序的 `java` 命令下添加参数

```sh
-Dsun.misc.ProxyGenerator.saveGeneratedFiles=true`
```

至于生成类将打印在哪，首先必然是在项目所属的目录下。有上一节中构造生成类全限定名的内容，根据包名的规则去寻找即可。(具体不再详述)

直接通过 IDEA, Eclipse 或者其它工具，对生成类 .class 文件进行反编译可以查看到整个生成类的内容。

```java
package com.sun.proxy;

import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Method;
import java.lang.reflect.Proxy;
import java.lang.reflect.UndeclaredThrowableException;
import me.fangfeng.jdk.proxy.Code;
import me.fangfeng.jdk.proxy.ICodeFactory;

public final class $Proxy0 extends Proxy implements ICodeFactory {
    private static Method m1;
    private static Method m2;
    private static Method m4;
    private static Method m3;
    private static Method m0;

    public $Proxy0(InvocationHandler var1) throws  {
        super(var1);
    }

    public final boolean equals(Object var1) throws  {
        try {
            return (Boolean)super.h.invoke(this, m1, new Object[]{var1});
        } catch (RuntimeException | Error var3) {
            throw var3;
        } catch (Throwable var4) {
            throw new UndeclaredThrowableException(var4);
        }
    }

    public final String toString() throws  {
        try {
            return (String)super.h.invoke(this, m2, (Object[])null);
        } catch (RuntimeException | Error var2) {
            throw var2;
        } catch (Throwable var3) {
            throw new UndeclaredThrowableException(var3);
        }
    }

    public final Code getCode() throws  {
        try {
            return (Code)super.h.invoke(this, m4, (Object[])null);
        } catch (RuntimeException | Error var2) {
            throw var2;
        } catch (Throwable var3) {
            throw new UndeclaredThrowableException(var3);
        }
    }

    public final void setCode(Code var1) throws  {
        try {
            super.h.invoke(this, m3, new Object[]{var1});
        } catch (RuntimeException | Error var3) {
            throw var3;
        } catch (Throwable var4) {
            throw new UndeclaredThrowableException(var4);
        }
    }

    public final int hashCode() throws  {
        try {
            return (Integer)super.h.invoke(this, m0, (Object[])null);
        } catch (RuntimeException | Error var2) {
            throw var2;
        } catch (Throwable var3) {
            throw new UndeclaredThrowableException(var3);
        }
    }

    static {
        try {
            m1 = Class.forName("java.lang.Object").getMethod("equals", Class.forName("java.lang.Object"));
            m2 = Class.forName("java.lang.Object").getMethod("toString");
            m4 = Class.forName("me.fangfeng.jdk.proxy.ICodeFactory").getMethod("getCode");
            m3 = Class.forName("me.fangfeng.jdk.proxy.ICodeFactory").getMethod("setCode", Class.forName("me.fangfeng.jdk.proxy.Code"));
            m0 = Class.forName("java.lang.Object").getMethod("hashCode");
        } catch (NoSuchMethodException var2) {
            throw new NoSuchMethodError(var2.getMessage());
        } catch (ClassNotFoundException var3) {
            throw new NoClassDefFoundError(var3.getMessage());
        }
    }
}
```


```plain
  __                    __                  
 / _| __ _ _ __   __ _ / _| ___ _ __   __ _ 
| |_ / _` | '_ \ / _` | |_ / _ \ '_ \ / _` |
|  _| (_| | | | | (_| |  _|  __/ | | | (_| |
|_|  \__,_|_| |_|\__, |_|  \___|_| |_|\__, |
                 |___/                |___/ 
```

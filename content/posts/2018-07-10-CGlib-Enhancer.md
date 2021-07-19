---
title: CGlib Enhancer 主流程源码解析
author: fangfeng
date: 2018-07-10
categories:
  - 技术
tags:
  - CGlib
  - ASM
---

## 前言

此博文写作的目的: 
  - (Core) 通过了解 CGlib Enhancer 的整个调用链，了解其对于唯一依赖的 ASM 库的调用方式。
  - 基于 Enhancer 对已有字节码进行增强的进一步理解与掌握。

<!--more-->

## Enhancer

从这篇不是官方但更胜于官方文档的 [CGlib Guide](https://mydailyjava.blogspot.com/2013/11/cglib-missing-manual.html) 来看，它首先提到的第一个类就是 Enhancer。
其被用于创建 Java 代理类，相较于 Java 标准库的 Proxy 类，它**特别地**，能够支持那些没有实现接口的类的代理工作。

### Enhancer 简单示例展示

针对现有的 SampleClass 类
```java
public class SampleClass {
    public String test(String input) {
        return "Hello World!";
    }
}
```

使用 CGlib 的 Enhancer 对 SampleClass 进行增强，下列操作的目的在于将 test(String) 方法的返回值改写成 "Hello cglib!"

```java
@Test
public void testFixedValue() {
    // new 一个 Enhancer 实例
    Enhancer enhancer = new Enhancer();
    // 声明使用的父类是 SampleClass
    enhancer.setSuperclass(SampleClass.class);
    // 设置回调方法 - 回调方法实现为 FixedValue (固定值) .
    enhancer.setCallback(new FixedValue() {
        public Object loadObject() throws Exception {
            return "Hello cglib!";
        }
    });
    // 创建 SampleClass 的代理子类实例
    SampleClass proxy = (SampleClass) enhancer.create();
    Assert.assertEquals("Hello cglib!", proxy.test(null));
}
```

简单测试上述代码，可以了解到确实，代理类的 test(String) 方法的返回值被改写了。
那么，究竟这种操作是如何实现的？下面👇将进行具体的探究。

## 调用链跟踪

### 高度抽象的时序图

![Enhancer 调用链 时序图](https://img.ffutop.com/11657CC2-D2A9-4ADD-89B8-6359C39C6181.jpg)

*下列内容将根据时序图进行组织，根据调用编号(1,2,3...)进行展开*

### Seq 1.

```java
public Object create() {
    classOnly = false;
    argumentTypes = null;
    return createHelper();
}

public Object create(Class[] argumentTypes, Object[] arguments) {
    classOnly = false;
    if (argumentTypes == null || arguments == null || argumentTypes.length != arguments.length) {
        throw new IllegalArgumentException("Arguments must be non-null and of equal length");
    }
    this.argumentTypes = argumentTypes;
    this.arguments = arguments;
    return createHelper();
}
```

上述两个 `create(...)` 方法。结合上一节的使用示例，可以看到 `create()` 对应的**无参构造**。
存在无参构造，那么**有参的构造方法**显然也是应该被支持id。`create(Class[], Object[])` 正是为了这个目标而存在。通过提供参数类型数组和参数实例数组，可以唯一地定位及调用代理类的特定构造方法。

这个方法比较简单，只是将 Enhancer 作为一个数据内容的持有者，重置了新代理类将使用的构造函数参数内容，之后就直接调用了 `createHelper()` 。

### Seq 2.

```java
private Object createHelper() {
    // 对预声明的回调方法进行验证，要求 [多个回调就必须存在 CallbackFilter 作为调度器]
    preValidate();
    // 构建一个对这类增强操作唯一定位的 key
    Object key = KEY_FACTORY.newInstance((superclass != null) ? superclass.getName() : null,
            ReflectUtils.getNames(interfaces),
            filter == ALL_ZERO ? null : new WeakCacheKey<CallbackFilter>(filter),
            callbackTypes,
            useFactory,
            interceptDuringConstruction,
            serialVersionUID);
    this.currentKey = key;
    // 调用父类通过的 create(...) 方法
    Object result = super.create(key);
    return result;
}
```

### Seq 3.

```java
protected Object create(Object key) {
    try {
        // 获取用于 加载 生成类 的 ClassLoader
        ClassLoader loader = getClassLoader();
        // 从缓存中加载 这个 ClassLoader 过去加载的相关数据
        Map<ClassLoader, ClassLoaderData> cache = CACHE;
        ClassLoaderData data = cache.get(loader);
        // 如果在 缓存 中找不到，则初始化构造关于这个 ClassLoader 的数据实例
        if (data == null) {
            // 同步
            synchronized (AbstractClassGenerator.class) {
                // 进入同步块后的 再次确认，避免重复初始化构建
                cache = CACHE;
                data = cache.get(loader);
                if (data == null) {
                    // 构建新的 缓存，拷贝原有的缓存集的内容
                    Map<ClassLoader, ClassLoaderData> newCache = new WeakHashMap<ClassLoader, ClassLoaderData>(cache);
                    // 初始化 ClassLoaderData ，真正的构造操作
                    data = new ClassLoaderData(loader);
                    // 添加到缓存中
                    newCache.put(loader, data);
                    CACHE = newCache;
                }
            }
        }
        this.key = key;
        Object obj = data.get(this, getUseCache());
        if (obj instanceof Class) {
            // 初次实例化操作，就是 Class 利用反射来进行实例化
            return firstInstance((Class) obj);
        }
        // 非初次实例化，则是从 ClassLoaderData 中得到的之前维护的内容
        return nextInstance(obj);
    } catch (RuntimeException e) {
        throw e;
    } catch (Error e) {
        throw e;
    } catch (Exception e) {
        throw new CodeGenerationException(e);
    }
}
```

这部分内容比较多，且是调用链比较重要的一环。**Seq 4.** 和 **Seq 5.** 将作为其子内容进行调用，但为了本博文的结构完整， *Seq 4. & Seq5.* 的标题与 *Seq 3.* 标题同级

### Seq 4. 

首先应该认识到，每个被加载的 Class ，在 `equal` 的判断过程中，依据的是同一个 ClassLoader 加载的同一 Class 才被认为是相同的，否则即使 Class 是同一个，但仍会被认为是不同的(由不同的 ClassLoader 加载)。

```java
public ClassLoader getClassLoader() {
    ClassLoader t = classLoader;
    if (t == null) {
        t = getDefaultClassLoader();
    }
    if (t == null) {
        t = getClass().getClassLoader();
    }
    if (t == null) {
        t = Thread.currentThread().getContextClassLoader();
    }
    if (t == null) {
        throw new IllegalStateException("Cannot determine classloader");
    }
    return t;
}
```

因此，在这部分中，首先需要确定的是由哪个 ClassLoader 进行加载的工作。`getClassLoader()` 的确定顺序是:

1. 具体实现类声明的 **默认 ClassLoader** 为第一优先级
2. 加载当前类(这里是 Enhancer) 的 ClassLoader 为第二优先级
3. **当前线程上下文 ClassLoader** 为第三优先级
4. 抛出异常

回到 **Seq 3.** 的内容，
下一步是对当前这个**AbstractClassGenerator** 维护的 CACHE 进行处理，如果在 CACHE 中能够得到以确定的 ClassLoader 为 Key 的键值对，则直接获取 ClassLoader 对应的值，否则将进行初始化构建一个新的 ClassLoaderData 作为这个 ClassLoader 的值。

### Seq 5.
在构建 ClassLoaderData 的过程中，最重要的一步:

```java
public ClassLoaderData(ClassLoader classLoader) {
    // ClassLoader 不可为空
    if (classLoader == null) {
        throw new IllegalArgumentException("classLoader == null is not yet supported");
    }
    // 构建 ClassLoader 的弱引用
    this.classLoader = new WeakReference<ClassLoader>(classLoader);
    // 写了个后置调用的函数，用于懒加载，只有调用了 load.apply(...) 才会构建 生成类
    Function<AbstractClassGenerator, Object> load =
            new Function<AbstractClassGenerator, Object>() {
                public Object apply(AbstractClassGenerator gen) {
                    Class klass = gen.generate(ClassLoaderData.this);
                    return gen.wrapCachedClass(klass);
                }
            };
    generatedClasses = new LoadingCache<AbstractClassGenerator, Object, Object>(GET_KEY, load);
}
```

构建了一个 Function 实例，这个 Function 作为函数式接口，预定义执行的具体流程，但由上下文确定具体的调用时刻。

### Seq 6.

在正确得到了 ClassLoader 对应的 ClassLoaderData 之后，ClassLoaderData 的 `get(...)` 方法将在特定的 ClassLoader 之下，进行更具体的加载新生成类的工作。
当然，这里的前提是新的生成类的字节码已经被构建:)

```java
public Object get(AbstractClassGenerator gen, boolean useCache) {
    // 标记为不使用缓存，直接构建 新的生成类
    if (!useCache) {
      return gen.generate(ClassLoaderData.this);
    }
    // 使用缓冲的情况
    else {
      Object cachedValue = generatedClasses.get(gen);
      return gen.unwrapCachedValue(cachedValue);
    }
}
```

可以看到 `gen.generate(ClassLoaderData.this)` 这段代码在 `get(...)` 方法和上一小节 `ClassLoaderData(...)` 构造方法的 Function 函数式实例都出现了。

实际上，这也是触发 ClassLoader 来加载动态生成的新 Class 的唯一入口。

*判断逻辑* 的 `else` 内容的唯一区别就在于其构建的是一个加载缓存，将真正触发 gen.generate(ClassLoaderData.this) 的逻辑延迟。(不过，最终也还是会进行调用的，:) 这是毋庸置疑的)。

### Seq 7.

```java
protected Class generate(ClassLoaderData data) {
    Class gen;
    Object save = CURRENT.get();
    CURRENT.set(this);
    try {
        // 拿到用于加载生成类的 ClassLoader
        ClassLoader classLoader = data.getClassLoader();
        if (classLoader == null) {
            throw new IllegalStateException("ClassLoader is null while trying to define class " +
                    getClassName() + ". It seems that the loader has been expired from a weak reference somehow. " +
                    "Please file an issue at cglib's issue tracker.");
        }
        // 构建一个合法的 生成类 的类名(非重复)
        synchronized (classLoader) {
          String name = generateClassName(data.getUniqueNamePredicate());
          data.reserveName(name);
          this.setClassName(name);
        }
        if (attemptLoad) {
            try {
                // 尝试直接通过 ClassLoader 进行加载
                gen = classLoader.loadClass(getClassName());
                return gen;
            } catch (ClassNotFoundException e) {
                // ignore
            }
        }
        // 策略下的生成类构建方法
        byte[] b = strategy.generate(this);
        // 通过解析字节码的形式获取 生成类的 className
        String className = ClassNameReader.getClassName(new ClassReader(b));
        ProtectionDomain protectionDomain = getProtectionDomain();
        synchronized (classLoader) { // just in case
            // 反射的形式加载 Class 类
            if (protectionDomain == null) {
                gen = ReflectUtils.defineClass(className, b, classLoader);
            } else {
                gen = ReflectUtils.defineClass(className, b, classLoader, protectionDomain);
            }
        }
        return gen;
    } catch (RuntimeException e) {
        throw e;
    } catch (Error e) {
        throw e;
    } catch (Exception e) {
        throw new CodeGenerationException(e);
    } finally {
        CURRENT.set(save);
    }
}
```

这个方法，就是操作字节码，加载 Class 的核心调度方法。

可以看到 `generateClassName(...)` 方法将构建一个当前 ClassLoader 下唯一, 不重复的新的类名。

`strategy.generate(this)` 将通过特定策略实现的形式生成新的字节码

`ReflectUtils.defineClass(className, b, classLoader)` 将使用反射使得 ClassLoader 来加载这个新的生成类。

### Seq 8.

生成新的不重复类名这部分不做过多陈述，可以简单进行了解，并熟悉 CGlib 默认的新类名的命名规则

```java
// DefaultNamePolicy
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

### Seq 9.

```java
// 下列是 DefaultGeneratorStrategy 的实现
public byte[] generate(ClassGenerator cg) throws Exception {
    DebuggingClassWriter cw = getClassVisitor();
    transform(cg).generateClass(cw);
    return transform(cw.toByteArray());
}
```

### Seq 10.

调用 `generateClass(ClassVisitor)` 将获得到新类的字节码。

```java
public void generateClass(ClassVisitor v) throws Exception {
    // 确定生成类的 父类
    Class sc = (superclass == null) ? Object.class : superclass;

    // 父类标识符不可以为 final
    if (TypeUtils.isFinal(sc.getModifiers()))
        throw new IllegalArgumentException("Cannot subclass final class " + sc.getName());
    // 获取父类直接声明的构造方法
    List constructors = new ArrayList(Arrays.asList(sc.getDeclaredConstructors()));
    filterConstructors(sc, constructors);

    // Order is very important: must add superclass, then
    // its superclass chain, then each interface and
    // its superinterfaces.
    List actualMethods = new ArrayList();
    List interfaceMethods = new ArrayList();
    final Set forcePublic = new HashSet();
    // 从父类中提取各种信息
    getMethods(sc, interfaces, actualMethods, interfaceMethods, forcePublic);

    List methods = CollectionUtils.transform(actualMethods, new Transformer() {
        public Object transform(Object value) {
            Method method = (Method)value;
            int modifiers = Constants.ACC_FINAL
                | (method.getModifiers()
                   & ~Constants.ACC_ABSTRACT
                   & ~Constants.ACC_NATIVE
                   & ~Constants.ACC_SYNCHRONIZED);
            if (forcePublic.contains(MethodWrapper.create(method))) {
                modifiers = (modifiers & ~Constants.ACC_PROTECTED) | Constants.ACC_PUBLIC;
            }
            return ReflectUtils.getMethodInfo(method, modifiers);
        }
    });

    ClassEmitter e = new ClassEmitter(v);
    if (currentData == null) {
    e.begin_class(Constants.V1_2,
                  Constants.ACC_PUBLIC,
                  getClassName(),
                  Type.getType(sc),
                  (useFactory ?
                   TypeUtils.add(TypeUtils.getTypes(interfaces), FACTORY) :
                   TypeUtils.getTypes(interfaces)),
                  Constants.SOURCE_FILE);
    } else {
        e.begin_class(Constants.V1_2,
                Constants.ACC_PUBLIC,
                getClassName(),
                null,
                new Type[]{FACTORY},
                Constants.SOURCE_FILE);
    }
    List constructorInfo = CollectionUtils.transform(constructors, MethodInfoTransformer.getInstance());

    e.declare_field(Constants.ACC_PRIVATE, BOUND_FIELD, Type.BOOLEAN_TYPE, null);
    e.declare_field(Constants.ACC_PUBLIC | Constants.ACC_STATIC, FACTORY_DATA_FIELD, OBJECT_TYPE, null);
    if (!interceptDuringConstruction) {
        e.declare_field(Constants.ACC_PRIVATE, CONSTRUCTED_FIELD, Type.BOOLEAN_TYPE, null);
    }
    e.declare_field(Constants.PRIVATE_FINAL_STATIC, THREAD_CALLBACKS_FIELD, THREAD_LOCAL, null);
    e.declare_field(Constants.PRIVATE_FINAL_STATIC, STATIC_CALLBACKS_FIELD, CALLBACK_ARRAY, null);
    if (serialVersionUID != null) {
        e.declare_field(Constants.PRIVATE_FINAL_STATIC, Constants.SUID_FIELD_NAME, Type.LONG_TYPE, serialVersionUID);
    }

    for (int i = 0; i < callbackTypes.length; i++) {
        e.declare_field(Constants.ACC_PRIVATE, getCallbackField(i), callbackTypes[i], null);
    }
    // This is declared private to avoid "public field" pollution
    e.declare_field(Constants.ACC_PRIVATE | Constants.ACC_STATIC, CALLBACK_FILTER_FIELD, OBJECT_TYPE, null);

    if (currentData == null) {
        emitMethods(e, methods, actualMethods);
        emitConstructors(e, constructorInfo);
    } else {
        emitDefaultConstructor(e);
    }
    emitSetThreadCallbacks(e);
    emitSetStaticCallbacks(e);
    emitBindCallbacks(e);

    if (useFactory || currentData != null) {
        int[] keys = getCallbackKeys();
        emitNewInstanceCallbacks(e);
        emitNewInstanceCallback(e);
        emitNewInstanceMultiarg(e, constructorInfo);
        emitGetCallback(e, keys);
        emitSetCallback(e, keys);
        emitGetCallbacks(e);
        emitSetCallbacks(e);
    }

    e.end_class();
}
```

`generateClass(...)` 执行的操作和 ASM 库的 ClassVisitor 访问 Class 文件结构的流程基本类似。

从上述截取到的部分代码，例如:

```java
e.begin_class(Constants.V1_2, Constants.ACC_PUBLIC, getClassName(), Type.getType(sc), 
        (useFactory ? TypeUtils.add(TypeUtils.getTypes(interfaces), FACTORY) : TypeUtils.getTypes(interfaces)),
        Constants.SOURCE_FILE);
```

```java
e.declare_field(Constants.PRIVATE_FINAL_STATIC, THREAD_CALLBACKS_FIELD, THREAD_LOCAL, null);
```

都与 `classVisitor.visit(...)` 以及 `classVisitor.visitField(...)` 相当类似。而事实上，这些方法的具体实现也确实调用了 cv.visitXxx(...) 。
毕竟，作为 CGlib 唯一依赖的库，ASM 还是可以说是难以替代的(既然已经有了可用的轮子，又何必重复造呢?)

## 构造的实例

下面将展示通过 CGlib 生成的新的类(由于生成的是字节码，懒得再自行解析 ClassFile 的内容，所有直接用 `IDEA` 做了字节码的解析)

**首先展示的需要进行增强的 SampleClass 的具体内容**
```java
public class SampleClass {
    public String test(String input) {
        return "Hello World!";
    }
}
```

**用于增强的简单代码**

```java
// 省略部分代码
public static void main(String... args) {
    Enhancer enhancer = new Enhancer();
    enhancer.setSuperclass(SampleClass.class);
    enhancer.setCallback(new FixedValue() {
        public Object loadObject() throws Exception {
            return "Hello cglib!";
        }
    });
    SampleClass proxy = (SampleClass) enhancer.create();
}
```

**动态生成的新的类**

```java
//
// Source code recreated from a .class file by IntelliJ IDEA
// (powered by Fernflower decompiler)
//

// 不用过多关注，只是 CGlib 保证生成类与原有的类一定属于同一包下，偷懒直接把 SampleClass 写在 CGlib 项目里了
package net.sf.cglib.samples;

import net.sf.cglib.proxy.Callback;
import net.sf.cglib.proxy.Factory;
import net.sf.cglib.proxy.FixedValue;

/**
 * 新的类名，通过 $$EnhancerByCGLIB 以及 $$7cd64b81(这个是哈希值) 对类名进行唯一区分 
 * 如果还是重复了，在动态生成前会进行类名的检测，并通过增加序号 "_<index>" 的形式进行进一步区分
 *
 * 可以看到新生成的类继承了 SampleClass 
 */
public class SampleClass$$EnhancerByCGLIB$$7cd64b81 extends SampleClass implements Factory {
    private boolean CGLIB$BOUND;
    public static Object CGLIB$FACTORY_DATA;
    private static final ThreadLocal CGLIB$THREAD_CALLBACKS;
    private static final Callback[] CGLIB$STATIC_CALLBACKS;

    // 绑定一个在增强中声明的 Callback 实例
    private FixedValue CGLIB$CALLBACK_0;
    // 绑定一个静态的回调调度实例
    private static Object CGLIB$CALLBACK_FILTER;

    static void CGLIB$STATICHOOK1() {
        CGLIB$THREAD_CALLBACKS = new ThreadLocal();
    }

    /**
      * 对 test(String) 的方法的增强
      */ 
    public final String test(String var1) {
        // 在方法块中拿到 Callback 实例
        FixedValue var10000 = this.CGLIB$CALLBACK_0;
        // 为空则尝试获取
        if (this.CGLIB$CALLBACK_0 == null) {
            CGLIB$BIND_CALLBACKS(this);
            var10000 = this.CGLIB$CALLBACK_0;
        }

        // 触发回调实例的方法获得返回值
        return (String)var10000.loadObject();
    }

    /**
      * 由于此次增强只声明了一个 Callback
      * 因此所有方法的增强都相同, 都是调用这个回调方法获取
      */
    public final boolean equals(Object var1) {
        FixedValue var10000 = this.CGLIB$CALLBACK_0;
        if (this.CGLIB$CALLBACK_0 == null) {
            CGLIB$BIND_CALLBACKS(this);
            var10000 = this.CGLIB$CALLBACK_0;
        }

        Object var2 = var10000.loadObject();
        /**
         * 但是，此处会尝试强制转型
         * 同时在最终执行失败的时候直接抛出运行时异常
         * 毕竟上面增强的时候返回固定值 "Hello, cglib!"
         */
        return var2 == null ? false : (Boolean)var2;
    }

    public final String toString() {
        FixedValue var10000 = this.CGLIB$CALLBACK_0;
        if (this.CGLIB$CALLBACK_0 == null) {
            CGLIB$BIND_CALLBACKS(this);
            var10000 = this.CGLIB$CALLBACK_0;
        }

        return (String)var10000.loadObject();
    }

    public final int hashCode() {
        FixedValue var10000 = this.CGLIB$CALLBACK_0;
        if (this.CGLIB$CALLBACK_0 == null) {
            CGLIB$BIND_CALLBACKS(this);
            var10000 = this.CGLIB$CALLBACK_0;
        }

        Object var1 = var10000.loadObject();
        return var1 == null ? 0 : ((Number)var1).intValue();
    }

    protected final Object clone() throws CloneNotSupportedException {
        FixedValue var10000 = this.CGLIB$CALLBACK_0;
        if (this.CGLIB$CALLBACK_0 == null) {
            CGLIB$BIND_CALLBACKS(this);
            var10000 = this.CGLIB$CALLBACK_0;
        }

        return var10000.loadObject();
    }

    public SampleClass$$EnhancerByCGLIB$$7cd64b81() {
        CGLIB$BIND_CALLBACKS(this);
    }

    public static void CGLIB$SET_THREAD_CALLBACKS(Callback[] var0) {
        CGLIB$THREAD_CALLBACKS.set(var0);
    }

    public static void CGLIB$SET_STATIC_CALLBACKS(Callback[] var0) {
        CGLIB$STATIC_CALLBACKS = var0;
    }

    private static final void CGLIB$BIND_CALLBACKS(Object var0) {
        SampleClass$$EnhancerByCGLIB$$7cd64b81 var1 = (SampleClass$$EnhancerByCGLIB$$7cd64b81)var0;
        if (!var1.CGLIB$BOUND) {
            var1.CGLIB$BOUND = true;
            Object var10000 = CGLIB$THREAD_CALLBACKS.get();
            if (var10000 == null) {
                var10000 = CGLIB$STATIC_CALLBACKS;
                if (CGLIB$STATIC_CALLBACKS == null) {
                    return;
                }
            }

            var1.CGLIB$CALLBACK_0 = (FixedValue)((Callback[])var10000)[0];
        }

    }

    public Object newInstance(Callback[] var1) {
        CGLIB$SET_THREAD_CALLBACKS(var1);
        SampleClass$$EnhancerByCGLIB$$7cd64b81 var10000 = new SampleClass$$EnhancerByCGLIB$$7cd64b81();
        CGLIB$SET_THREAD_CALLBACKS((Callback[])null);
        return var10000;
    }

    public Object newInstance(Callback var1) {
        CGLIB$SET_THREAD_CALLBACKS(new Callback[]{var1});
        SampleClass$$EnhancerByCGLIB$$7cd64b81 var10000 = new SampleClass$$EnhancerByCGLIB$$7cd64b81();
        CGLIB$SET_THREAD_CALLBACKS((Callback[])null);
        return var10000;
    }

    public Object newInstance(Class[] var1, Object[] var2, Callback[] var3) {
        CGLIB$SET_THREAD_CALLBACKS(var3);
        SampleClass$$EnhancerByCGLIB$$7cd64b81 var10000 = new SampleClass$$EnhancerByCGLIB$$7cd64b81;
        switch(var1.length) {
        case 0:
            var10000.<init>();
            CGLIB$SET_THREAD_CALLBACKS((Callback[])null);
            return var10000;
        default:
            throw new IllegalArgumentException("Constructor not found");
        }
    }

    public Callback getCallback(int var1) {
        CGLIB$BIND_CALLBACKS(this);
        FixedValue var10000;
        switch(var1) {
        case 0:
            var10000 = this.CGLIB$CALLBACK_0;
            break;
        default:
            var10000 = null;
        }

        return var10000;
    }

    public void setCallback(int var1, Callback var2) {
        switch(var1) {
        case 0:
            this.CGLIB$CALLBACK_0 = (FixedValue)var2;
        default:
        }
    }

    public Callback[] getCallbacks() {
        CGLIB$BIND_CALLBACKS(this);
        return new Callback[]{this.CGLIB$CALLBACK_0};
    }

    public void setCallbacks(Callback[] var1) {
        this.CGLIB$CALLBACK_0 = (FixedValue)var1[0];
    }

    static {
        CGLIB$STATICHOOK1();
    }
}
```

从上面的生成类可以看出来，CGlib 所进行的增强，是在不直接对原有的方法体内容进行改造的基础上(例如上面的 test(String) 方法，没有直接改写成 `return "Hello, cglib!` 的形式)，对方法执行之前入参或者方法执行之后的返回值进行一定的处理。

主要使用的形式就是将回调实例与原有方法进行绑定，并通过调用回调方法来实现方法的增强。
```plain
  __                    __                  
 / _| __ _ _ __   __ _ / _| ___ _ __   __ _ 
| |_ / _` | '_ \ / _` | |_ / _ \ '_ \ / _` |
|  _| (_| | | | | (_| |  _|  __/ | | | (_| |
|_|  \__,_|_| |_|\__, |_|  \___|_| |_|\__, |
                 |___/                |___/ 
```

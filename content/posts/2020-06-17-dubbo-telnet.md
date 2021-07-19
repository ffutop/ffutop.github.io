---
title: Dubbo Telnet 调试
author: fangfeng
date: 2020-06-17
tags:
  - Dubbo
  - Telnet
  - Java
categories:
  - Cheat Sheet
---

始于 Dubbo 2.0.6 的 Telnet Command 是一个令人兴奋的特性，极大地降低了服务化测试的成本，但是，[寥寥数行的可怜文档](http://dubbo.apache.org/zh-cn/docs/user/references/telnet.html)无形地为使用增加了成本。此前虽然一直在使用 Telnet Command，但基本上是浅尝辄止，字符集的问题、重载方法的错误筛选等，都让我不得不对这个特性敬而远之，无法作为高频的生产力工具。最近，频繁出现的调试需求让我不得不尝试接受并熟悉 Dubbo Telnet Command。

*本文只针对 invoke 命令，基于 Dubbo 版本 2.6.7*

Dubbo Telnet Command `invoke` 命令的一般格式为 `invoke <全限定名>.<方法名>(<参数>,...,<参数>)`。其中参数需要能被 JSON 解析，即提取命令中的 `<参数>,...,<参数>` 部分，并包装上 `[]` 构成 `[<参数>,...,<参数>]` ，需要保证这个串是一个合法的 JSON Array。

本文提供的示例均可在 [dubbo-telnet-playground](https://github.com/ffutop/dubbo-telnet-playground) 中找到。

<!--more-->

## 一、基本类型

八大基本类型，映射到 [JSON](https://www.json.org/json-en.html) 对应为 `number`, `"true"/"false"`。

```bash
telnet 127.0.0.1 20880
Trying 127.0.0.1...
Connected to localhost.
Escape character is '^]'.

# bool 类型
dubbo>invoke com.ffutop.playground.api.PrimitiveService.boolMethod(false)
"com.ffutop.playground.service.PrimitiveServiceImpl#boolMethod(false)"
elapsed: 2 ms.

# int 类型
dubbo>invoke com.ffutop.playground.api.PrimitiveService.intMethod(123456)
"com.ffutop.playground.service.PrimitiveServiceImpl#intMethod(123456)"
elapsed: 0 ms.

# byte 类型
dubbo>invoke com.ffutop.playground.api.PrimitiveService.byteMethod(1)
"com.ffutop.playground.service.PrimitiveServiceImpl#byteMethod(1)"
elapsed: 0 ms.

# short 类型
dubbo>invoke com.ffutop.playground.api.PrimitiveService.shortMethod(10000)
"com.ffutop.playground.service.PrimitiveServiceImpl#shortMethod(10000)"
elapsed: 0 ms.

# long 类型
dubbo>invoke com.ffutop.playground.api.PrimitiveService.longMethod(1234567892346754)
"com.ffutop.playground.service.PrimitiveServiceImpl#longMethod(1234567892346754)"
elapsed: 0 ms.

# double 类型
dubbo>invoke com.ffutop.playground.api.PrimitiveService.doubleMethod(1.0022)
"com.ffutop.playground.service.PrimitiveServiceImpl#doubleMethod(1.0022)"
elapsed: 0 ms.

# char 类型
dubbo>invoke com.ffutop.playground.api.PrimitiveService.charMethod('a')
"com.ffutop.playground.service.PrimitiveServiceImpl#charMethod(a)"
elapsed: 0 ms.

# float 类型
dubbo>invoke floatMethod(1.234)
"com.ffutop.playground.service.PrimitiveServiceImpl#floatMethod(1.234)"
elapsed: 0 ms.
```

## 二、引用类型

引用类型参数一般使用一个 JSON object 来描述。object 中一般使用 `"class":"xxx"` 强制声明 object 对应的 Java 对象类型，便于参数筛选和对象实例化。

其中特殊的一个是 `java.lang.String` ，可以由 JSON string 直接描述，当然也可以选择用 object 来进行描述。

```java
public class PlayModel {
    int intParam;
    double doubleParam;
    String stringParam;
    // 省略 get/set 方法
}
```

```bash
# 罗列 com.ffutop.playground.api.ReferenceService 所有的方法
dubbo>ls -l com.ffutop.playground.api.ReferenceService
java.lang.String refMethod(com.ffutop.playground.model.PlayModel)
java.lang.String stringMethod(java.lang.String)

# 自定义 OBJECT 引用类型
dubbo>invoke com.ffutop.playground.api.ReferenceService.refMethod({"class":"com.ffutop.playground.model.PlayModel","intParam":1,"doubleParam":2.233,"stringParam":"HelloWorld"})
"com.ffutop.playground.service.ReferenceServiceImpl#refMethod({\"doubleParam\":2.233,\"intParam\":1,\"stringParam\":\"HelloWorld\"})"
elapsed: 14 ms.

# String 引用类型 - 第一类调用方案
dubbo>invoke com.ffutop.playground.api.ReferenceService.stringMethod("Hello World")
"com.ffutop.playground.service.ReferenceServiceImpl#stringMethod(\"Hello World\")"
elapsed: 4 ms.

# String 引用类型 - 第二类调用方案
dubbo>invoke com.ffutop.playground.api.ReferenceService.stringMethod({"class":"java.lang.String","value":"Hello World"})
"com.ffutop.playground.service.ReferenceServiceImpl#stringMethod(\"Hello World\")"
elapsed: 0 ms.
```

## 三、集合类型

常规的集合类型有 `列表(List)`，`集(Set)`，`映射(Map)`，这里把 `数组(Arrays)` 也算在其中。

Arrays, List, Map 可以分别有 JSON 的 array, array, object 来描述。

只有 Set 比较特殊，JSON 中没有直接描述 Set 的结构，只能借用 JSON object 来描述一个 `HashSet` 对象。`{"class":"java.util.HashSet","map":{"Hello":null,"World":null}}`，`class` 标识 object 需要实例化的 Java 对象类，`map` 是 `HashSet` 中的字段，将通过反射机制塞入需要最终实例化得到的 `HashSet` 对象。

```java
public class HashSet<E>
    extends AbstractSet<E>
    implements Set<E>, Cloneable, java.io.Serializable
{
    static final long serialVersionUID = -5024744406713321676L;

    private transient HashMap<E,Object> map;

    // More Code Omitted
}
```

```bash
dubbo>ls -l com.ffutop.playground.api.CollectionService
java.lang.String setMethod(java.util.Set)
java.lang.String arrayMethod(java.lang.Object[])
java.lang.String listMethod(java.util.List)
java.lang.String mapMethod(java.util.Map)

# 数组(Arrays)类型
dubbo>invoke arrayMethod([1293,234.23,"@#(@#JNE",{"class":"com.ffutop.playground.model.PlayModel","intParam":123,"stringParam":"HelloWorld"}])
"com.ffutop.playground.service.OverloadServiceImpl#arrayMethod([1293,234.23,\"@#(@#JNE\",{\"doubleParam\":0.0,\"intParam\":123,\"stringParam\":\"HelloWorld\"}])"
elapsed: 5 ms.

# 列表(List)类型
dubbo>invoke listMethod([1293,234.23,"@#(@#JNE",{"class":"com.ffutop.playground.model.PlayModel","intParam":123,"stringParam":"HelloWorld"}])
"com.ffutop.playground.service.CollectionServiceImpl#listMethod([1293,234.23,\"@#(@#JNE\",{\"doubleParam\":0.0,\"intParam\":123,\"stringParam\":\"HelloWorld\"}])"
elapsed: 0 ms.

# 集(Set)类型
dubbo>invoke setMethod({"class":"java.util.HashSet","map":{"Hello":null,"World":null}})
"com.ffutop.playground.service.CollectionServiceImpl#setMethod([\"Hello\",\"World\"])"
elapsed: 0 ms.

# 映射(Map)类型
dubbo>invoke mapMethod({"class":"java.util.HashMap","Hello":123,"World":456})
"com.ffutop.playground.service.CollectionServiceImpl#mapMethod({\"Hello\":123,\"World\":456})"
elapsed: 0 ms.
```

## 四、多参数方法

介绍过多种典型的参数类型之后，接着需要了解多参数方法的调用方式。其实也比较简单，不过是按照 Dubbo 接口方法中参数的声明顺序，将每个单参数 JSON 串整合，并通过 `,` 分割即可。

```bash
dubbo>ls -l com.ffutop.playground.api.MultiParamsService
java.lang.String multiParamsMethod2(java.util.Map,java.util.List)
java.lang.String multiParamsMethod3(int,int,int,java.lang.String)
java.lang.String multiParamsMethod1(int,com.ffutop.playground.model.PlayModel)

dubbo>invoke multiParamsMethod1(1781687,{"class":"com.ffutop.playground.model.PlayModel","intParam":1781688,"doubleParam":178.1688,"stringParam":"Hello World"})
"com.ffutop.playground.service.MultiParamsServiceImpl#multiParamsMethod1(1781687,{\"doubleParam\":178.1688,\"intParam\":1781688,\"stringParam\":\"Hello World\"})"
elapsed: 11 ms.

dubbo>invoke multiParamsMethod2({"p1":"HELLO","p2":178.1688},["hello","world","1314"])
"com.ffutop.playground.service.MultiParamsServiceImpl#multiParamsMethod2({\"p1\":\"HELLO\",\"p2\":178.1688},[\"hello\",\"world\",\"1314\"])"
elapsed: 2 ms.

dubbo>invoke multiParamsMethod3(178,1687,1688,"Hello World")
"com.ffutop.playground.service.MultiParamsServiceImpl#multiParamsMethod3(178,1687,1688,Hello World)"
elapsed: 1 ms.
```

## 五、重载方法

虽然 Dubbo Telnet Command 也支持重载方法，但存在 BUG ，对于一些特定的重载方法，将发生 `No Service` 或者调用错误的方法等问题。

Dubbo Telnet Command - Invoke 的核心逻辑包含两个部分，其一是基于用户输入，筛选合适的 Dubbo provider 及方法；其二是对用户输入进行反序列化，构造 Java 对象并最终触发执行。

对于几种 Java 基本类型以及 `String`，由于 JSON 提供的描述信息及差异不足，对于此类重载，将在筛选方法时，产生错误选择，并最终导致反序列化失败。当然，另一方面这也可以被认定为 Dubbo Telnet 的 BUG。因此，当重载方法入参数量相同，且都是基本类型/`String`，则将无法调用成功。

```bash
dubbo>ls -l com.ffutop.playground.api.OverloadService
java.lang.String method(com.ffutop.playground.model.PlayModel)
java.lang.String method(com.ffutop.playground.model.AnotherPlayModel)
java.lang.String method(java.lang.String,com.ffutop.playground.model.PlayModel)
java.lang.String method(com.ffutop.playground.model.PlayModel,com.ffutop.playground.model.AnotherPlayModel)
java.lang.String method(java.lang.String)
java.lang.String method(java.lang.Integer)
java.lang.String method(long)

invoke method(178)
Failed to invoke method method, cause: java.lang.ClassCastException: java.lang.Integer cannot be cast to java.lang.String
java.lang.ClassCastException: java.lang.Integer cannot be cast to java.lang.String
	at com.alibaba.dubbo.common.bytecode.Wrapper1.invokeMethod(Wrapper1.java)
    # something omitted

dubbo>invoke method("1244")
"com.ffutop.playground.service.OverloadServiceImpl#method(String 1244)"
elapsed: 1 ms.

dubbo>invoke method({"class":"com.ffutop.playground.model.PlayModel","intParam":178,"doubleParam":0.1688,"stringParam":"HELLO WORLD"})
"com.ffutop.playground.service.OverloadServiceImpl#method(PlayModel {\"doubleParam\":0.1688,\"intParam\":178,\"stringParam\":\"HELLO WORLD\"})"
elapsed: 10 ms.

dubbo>invoke method({"class":"com.ffutop.playground.model.AnotherPlayModel","longParam":1781688,"floatParam":167.1688,"playParam":{"class":"com.ffutop.playground.model.PlayModel","intParam":178,"doubleParam":0.1688,"stringParam":"HELLO WORLD"}})
"com.ffutop.playground.service.OverloadServiceImpl#method(AnotherPlayModel {\"floatParam\":167.1688,\"longParam\":1781688,\"playParam\":{\"doubleParam\":0.1688,\"intParam\":178,\"stringParam\":\"HELLO WORLD\"}})"
elapsed: 3 ms.
```

## 六、字符集问题解决

前述的所有方法面对的输入都是 ASCII 字符，对于几乎所有的字符集都是兼容的。但是，日常需求中**汉字**形式的入参并不少见，但多数尝试都被解析为乱码。

原因一般都是用户输入**汉字**所使用的字符集与 Dubbo Telnet Command 默认使用的字符集不相匹配。Dubbo 默认使用了 *GBK* ，对于习惯性使用 *UTF-8* 的开发人员来说，特别残酷。

不过这个问题的解决方法也特别简单，对用户输入输出做一个包装，完成字符集转化即可。下面提供一个 Python3 版本的 GBK:UTF-8 自转换的 Telnet 工具。

```python
#!/usr/bin/env python3
"""
Dubbo Tuned Telnet Command

Example:
    ./telent_dubbo.py localhost 20880
"""
import sys
import telnetlib
import platform
import readline

host = sys.argv[1]
port = sys.argv[2]
encoding = "GBK"

tn = telnetlib.Telnet(host, port)
cmd = None

while cmd != "exit":
    cmd = input("dubbo> ").strip()
    tn.write(cmd.encode(encoding) + b"\n")
    if cmd != "exit":
        result = tn.read_until(b"dubbo>").decode(encoding)
        result = result[:result.rfind('\n')]
        print(result)
```

## 写在最后

不得不说，除了文档方面的问题，Dubbo Telnet Command 解决了 Dubbo 服务调试的一个重大问题。


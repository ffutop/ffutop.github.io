---
title: ASM - ClassReader 与 Java ClassFile 文件格式
author: fangfeng
date: 2018-06-11
categories:
  - 技术
---

## Java ClassFile 文件格式

读了将近一周的时间，勉强算是把 ClassFile 的文件格式给简单的梳理了个脉络。
[The class File Format(Java SE 8)](https://docs.oracle.com/javase/specs/jvms/se8/html/index.html) 

> u1, u2, u4 分别表示无符号的一字节、二字节、四字节数据(以大端存储)

```c++
ClassFile {
    u4             magic;                                   // 魔数(magic) 固定为 0xCAFEBABE
    u2             minor_version;                           // 次版本号
    u2             major_version;                           // 主版本号
    u2             constant_pool_count;                     // 常量池 constant_pool 的数量 + 1, 最大为 (2<<16 - 1) = 65535
    cp_info        constant_pool[constant_pool_count-1];    // 常量池 取值下标为 [1, constant_pool_count)
    u2             access_flags;                            // 对类 or 接口的访问权限和属性的标志的掩码
    u2             this_class;                              // 值为 constant_pool 的有效下标(且 constant_pool[this_class] 的类型为 CONSTANT_Class_info)
    u2             super_class;                             // 值为 0 或者 constant_pool 的有效下标(同上), 如果是 interface, 则该值一定为有效下标
    u2             interfaces_count;                        // 直接父接口的数量
    u2             interfaces[interfaces_count];            // 值必须是 constant_pool 的有效下标, 且 interfaces[i](0≤i<interfaces_count), 指向的类型为 CONSTANT_Class_info)
    u2             fields_count;                            // 字段数量, 统计所有字段, 包括 class variables(静态变量) 和 instance variables(实例变量)
    field_info     fields[fields_count];                    // 字段的详细声明, 不包含继承来的字段
    u2             methods_count;                           // 方法数量
    method_info    methods[methods_count];                  // 方法的详细声明, 不包括继承来的方法
    u2             attributes_count;                        // 属性数量
    attribute_info attributes[attributes_count];            // 属性的详细声明
}
```
<!--more-->

通过 `javac Xxx.java` 命令得到的 `Xxx.class` 文件严格按照定义的 ClassFile 文件格式以8比特的字节为单位进行存储。

**下面的例子都将通过 Trie.java (实际代码见本文最后) 以及其经编译后的文件 Trie.class 做基本的介绍。**

### magic, minor_verion & major_version
通过命令 `xxd Trie.class` 查看 Trie.class 的十六进制编码，前16字节的内容如下:
```text
00000000: cafe babe 0000 0034 008f 0a00 2400 4b09  .......4....$.K.
```
可以看到前4字节的内容 `0xcafebabe`，没有实际意义，但是作为其是能否被JVM(Java Virtual Machine, Java虚拟机)接收的Class文件的一个校验。

紧接着的4个字节包括`次版本号` 和 `主版本号`，暂时不做深入。

### 常量池 constant_pool

从第10个字节开始(从第0字节开始计数)，连续的两个字节表示常量池中各种 CONSTANT 的总个数+1 (constant_pool_count)。constant_pool_count 只有2字节也就意味着: 常量池的最大常量数量为 (2<<16 - 1) = 65535 (当然，貌似在这个量级上，也从没见过有谁能够维护这样一个大JAVA类文件了)。

可以看到 constant_pool_count 的内容为 `0x008f = 143` ，即对于常量池的声明可以认为是 `cp_info constant_pool[142]`

```text
cp_info {
    u1 tag;
    u1 info[]; 
}
```

利用 JDK 自带的 `javap` 来查看 `Trie.class` 文件(`javap -v Trie.class`)，可以对 cp_info 的格式有一个粗浅但形象化的认识(结果见**附录2**)
其中第一列的内容 `#? = ?` `#?`表示id，`= ?`表示一个 cp_info 的实际类型，有 `cp_info.tag` 指定，具体映射表为

| Constant Type                 | Value |
| ----------------------------- | ----- |
| `CONSTANT_Class`              | 7     |
| `CONSTANT_Fieldref`           | 9     |
| `CONSTANT_Methodref`          | 10    |
| `CONSTANT_InterfaceMethodref` | 11    |
| `CONSTANT_String`             | 8     |
| `CONSTANT_Integer`            | 3     |
| `CONSTANT_Float`              | 4     |
| `CONSTANT_Long`               | 5     |
| `CONSTANT_Double`             | 6     |
| `CONSTANT_NameAndType`        | 12    |
| `CONSTANT_Utf8`               | 1     |
| `CONSTANT_MethodHandle`       | 15    |
| `CONSTANT_MethodType`         | 16    |
| `CONSTANT_InvokeDynamic`      | 18    |
| `CONSTANT_Module`             | 19    |
| `CONSTANT_Package`            | 20    |

在 Java ClassFile 的格式定义中，同时定义了每种 `CONSTANT` 的长度与格式。
例如:
```c++
CONSTANT_Class_info {
    u1 tag;
    u2 name_index;
}
```
```c++
CONSTANT_Fieldref_info {
     u1 tag;
     u2 class_index;
     u2 name_and_type_index;
}
CONSTANT_Methodref_info {
  u1 tag;
  u2 class_index;
  u2 name_and_type_index;
}
```
更多的 `CONSTANT_XXX` 的格式定义见 [ClassFile CONSTANT_XXX 结构](https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-4.html#jvms-4.4)

简单解析一下 cp_info[1]。
```text
00000000: cafe babe 0000 0034 008f 0a00 2400 4b09  .......4....$.K.

#1 = Methodref          #36.#75       // java/lang/Object."<init>":()V
```

`cp_info[1].tag = 0x0a = 10`，即 cp_info[1] 的类型为 Methodref 。
之后就直接套用 `CONSTANT_Methodref_info` 的数据结构定义的格式
`cp_info[1].class_index = 0x0024 = 36`，即 cp_info[1].class_index 表示的类名指向常量池第 36 个元素表示的类
`cp_info[1].name_and_type_index = 0x004b = 75`。表示指向的是常量池第 75 个元素表示的 NameAndType 结构

对于这部分内容的理解，如果觉得有能够阅读英文文档，可以参考 [The Java® Virtual Machine Specification Chap 4.](https://docs.oracle.com/javase/specs/jvms/se8/html/index.html)
否则，可以选择 [Java 虚拟机规范(Java SE 7 版) 第四章](https://icyfenix.iteye.com/blog/1256329)
并尝试根据经编译后.class文件的二进制码进行人工解析，以加深对此的理解。

### more... 

更多内容于此基本类似，均属于已经被严格预定义的规范化格式，对此进行解析即可得到相应的内容。

## ASM 概览

### 包结构

asm, asm-analysis, asm-commons, asm-tree, asm-util, asm-xml 模块基于 Maven 的组织形式，并包含有单元测试的内容。

asm-test 实现了对上述模块的单元测试的整合。

benchmarks 定义了一些基本的 JMH 基准测试来衡量ASM的表现

gradle 包含了一些 Gradle 包装器，来辅助 Linux or Windows 脚本的调用请求

tools 包含两个 Gradle 项目来帮助 ASM 生成 Artifacts 。

![ASM Structure](https://ws1.sinaimg.cn/large/006tKfTcgy1frywra01upj308g0akmxr.jpg)

### 代码组织形式

![Code Organization](https://asm.ow2.io/asm-package-dependencies.svg)

- org.objectweb.asm 作为核心包，定义了 ASM 访问者 API 以及两个核心类 ClassReader & ClassWriter 来读写编译后的 Java 类。该包不依赖于其他包，且可单独使用
- org.objectweb.asm.signature 提供了读写泛型签名的 API
- org.objectweb.asm.tree 提供了类 DOM API 与类 SAX API。
- org.objectweb.asm.tree.analysis 基于 .tree 提供了静态字节码分析框架
- org.objectweb.asm.commons 提供了一些基于 core 和 tree 包的类适配器。
- org.objectweb.asm.util 提供了一些用于 DEBUG 的访问者类与适配器类
- org.objectweb.asm.xml 目前已经被 @Deprecated 。提供了类文件与 XML 互相转换的能力

综上，asm 核心包是最为复杂的一个模块，而其它诸如 tree, util, xml 等都只是提供了将 JAVA 类从一种高级表示形态转换为另一种的能力。signature 也相当简单，提供了解析和打印简单语法的能力。

### 主要数据结构

核心包由 28 个类(接口) 组成。如果不考虑 Opcodes 接口, 5 个抽象访问者类(AnnotationVistitor, ClassVisitor, FieldVisitor, MethodVisitor and ModuleVisitor), 6 个工具类(ConstantDynamic, Contants, Handle, Type, TypePath and TypeReference), 剩下 16 个类的由下图提供一个粗略的展示。

![](https://asm.ow2.io/asm-package-overview.svg)

编译类到访问事件的转换只由一个类 ClassReader 以及辅助类 Context 完成。其逆过程由以 ClassWriter 为核心的其它 14 个类完成。


### ClassReader 

ClassReader 作为 ASM 核心包解析现有 .class 的唯一工具，组织形式相当简单。

- 在构造函数中完成对常量池和引导方法的解析
    - 存储每个常量池项目的起始偏移量 cpInfoOffsets
    - 存储每个引导方法的起始偏移量 bootstrapMethodOffsets
    - 存储最长字符串常量的大小 maxStringLength

```java
  ClassReader(final byte[] classFileBuffer, final int classFileOffset, final boolean checkClassVersion) {
    this.b = classFileBuffer;   // .class 文件缓存
    // 检查主版本号, 第6,7个字节(从0字节开始计数)
    if (checkClassVersion && readShort(classFileOffset + 6) > Opcodes.V11) {
      throw new IllegalArgumentException(
          "Unsupported class file major version " + readShort(classFileOffset + 6));
    }
    // 创建常量池数组，常量池长度的定义 constant_pool_count 在第8,9字节
    int constantPoolCount = readUnsignedShort(classFileOffset + 8);     // 读取无符号short, 即读取连续两字节作为一个short值
    cpInfoOffsets = new int[constantPoolCount];                         // 每个常量的偏移位置
    cpInfoValues = new Object[constantPoolCount];                       // 每个常量的实例对象
    int currentCpInfoIndex = 1;
    int currentCpInfoOffset = classFileOffset + 10;
    int currentMaxStringLength = 0;                                     // 最长字符串常量
    while (currentCpInfoIndex < constantPoolCount) {
      cpInfoOffsets[currentCpInfoIndex++] = currentCpInfoOffset + 1;
      int cpInfoSize;
      switch (classFileBuffer[currentCpInfoOffset]) {
        case Symbol.CONSTANT_FIELDREF_TAG:
        case Symbol.CONSTANT_METHODREF_TAG:
        case Symbol.CONSTANT_INTERFACE_METHODREF_TAG:
        case Symbol.CONSTANT_INTEGER_TAG:
        case Symbol.CONSTANT_FLOAT_TAG:
        case Symbol.CONSTANT_NAME_AND_TYPE_TAG:
        case Symbol.CONSTANT_INVOKE_DYNAMIC_TAG:
        case Symbol.CONSTANT_DYNAMIC_TAG:
          cpInfoSize = 5;
          break;
        case Symbol.CONSTANT_LONG_TAG:
        case Symbol.CONSTANT_DOUBLE_TAG:
          cpInfoSize = 9;
          currentCpInfoIndex++;
          break;
        case Symbol.CONSTANT_UTF8_TAG:
          cpInfoSize = 3 + readUnsignedShort(currentCpInfoOffset + 1);
          if (cpInfoSize > currentMaxStringLength) {
            // The size in bytes of this CONSTANT_Utf8 structure provides a conservative estimate
            // of the length in characters of the corresponding string, and is much cheaper to
            // compute than this exact length.
            currentMaxStringLength = cpInfoSize;
          }
          break;
        case Symbol.CONSTANT_METHOD_HANDLE_TAG:
          cpInfoSize = 4;
          break;
        case Symbol.CONSTANT_CLASS_TAG:
        case Symbol.CONSTANT_STRING_TAG:
        case Symbol.CONSTANT_METHOD_TYPE_TAG:
        case Symbol.CONSTANT_PACKAGE_TAG:
        case Symbol.CONSTANT_MODULE_TAG:
          cpInfoSize = 3;
          break;
        default:
          throw new IllegalArgumentException();
      }
      currentCpInfoOffset += cpInfoSize;
    }
    this.maxStringLength = currentMaxStringLength;
    // The Classfile's access_flags field is just after the last constant pool entry.
    this.header = currentCpInfoOffset;

    // 读取 BootstrapMethods 属性(如果存在)
    int currentAttributeOffset = getFirstAttributeOffset();
    int[] currentBootstrapMethodOffsets = null;
    for (int i = readUnsignedShort(currentAttributeOffset - 2); i > 0; --i) {
      // 读取每个 attribute_info 的属性名和属性长度
      String attributeName = readUTF8(currentAttributeOffset, new char[maxStringLength]);
      int attributeLength = readInt(currentAttributeOffset + 2);
      currentAttributeOffset += 6;
      // 如果当前属性名为 BootstrapMethods ，则进入处理逻辑
      if (Constants.BOOTSTRAP_METHODS.equals(attributeName)) {
        // Read the num_bootstrap_methods field and create an array of this size.
        currentBootstrapMethodOffsets = new int[readUnsignedShort(currentAttributeOffset)];
        // Compute and store the offset of each 'bootstrap_methods' array field entry.
        int currentBootstrapMethodOffset = currentAttributeOffset + 2;
        for (int j = 0; j < currentBootstrapMethodOffsets.length; ++j) {
          currentBootstrapMethodOffsets[j] = currentBootstrapMethodOffset;
          // Skip the bootstrap_method_ref and num_bootstrap_arguments fields (2 bytes each),
          // as well as the bootstrap_arguments array field (of size num_bootstrap_arguments * 2).
          currentBootstrapMethodOffset +=
              4 + readUnsignedShort(currentBootstrapMethodOffset + 2) * 2;
        }
      }
      currentAttributeOffset += attributeLength;
    }
    this.bootstrapMethodOffsets = currentBootstrapMethodOffsets;
  }
```

到此为止，ClassReader 已经解析了包括 magic, minor version, major version, constant pool 。
但是，诸如 field_info, method_info, attribute_info 等仍然没有得到处理。

这部分的内容在 accept(...) 和 readXXX(...) 中将得到解析。

主要流程类似:

1. 读取当前内容的偏移量(相较于整个 byte[])
2. 解析当前的内容
3. 调用 visitXXX 方法
4. 在 visitXXX 方法中进行相关的处理
5. visitEnd

## 附录1 Trie.java

```java
package me.fangfeng.filter;

import java.io.BufferedReader;
import java.io.BufferedWriter;
import java.io.FileInputStream;
import java.io.FileOutputStream;
import java.io.FileNotFoundException;
import java.io.InputStreamReader;
import java.io.IOException;
import java.io.OutputStreamWriter;

/**
 * @author fangfeng
 * @since 2018/5/16
 */
public class Trie {
    
    private static int MAX_ITEM = 700000;
    private static int AVG_LENGTH = 11;
    private static int MAX_NODE = MAX_ITEM * AVG_LENGTH;
    private static int CHAR_NUM = 10;

    int[][] nxt = new int[MAX_NODE][CHAR_NUM];
    boolean[] flag = new boolean[MAX_NODE];
    int trieIndex = 0;

    void insert(long number) {
        int tmpIndex = 0;
        for (;number != 0;number /= 10) {
            if (nxt[tmpIndex][(int) (number % 10)] == 0) {
                nxt[tmpIndex][(int) (number % 10)] = ++trieIndex;
            }
            tmpIndex = nxt[tmpIndex][(int) (number % 10)];
        }
        flag[tmpIndex] = true;
    }

    boolean query(long number) {
        int tmpIndex = 0;
        for (;number != 0;number /= 10) {
            if (nxt[tmpIndex][(int) (number % 10)] == 0) {
                return false;
            }
            tmpIndex = nxt[tmpIndex][(int) (number % 10)];
        }
        return flag[tmpIndex];
    }

    public static void main(String... args) throws FileNotFoundException, IOException {

        long start = System.currentTimeMillis();

        String ruleFilePath = args[0];
        String sendFilePath = args[1];
        String outFilePath = args[2];

        BufferedReader ruleReader = new BufferedReader(new InputStreamReader(new FileInputStream(ruleFilePath)));
        BufferedReader sendReader = new BufferedReader(new InputStreamReader(new FileInputStream(sendFilePath)));

        BufferedWriter outWriter = new BufferedWriter(new OutputStreamWriter(new FileOutputStream(outFilePath)));

        Trie trie = new Trie();
        String mobile;
        while((mobile = ruleReader.readLine()) != null) {
            trie.insert(Long.parseLong(mobile));
        }
        ruleReader.close();

        while((mobile = sendReader.readLine()) != null) {
            if(trie.query(Long.parseLong(mobile)) == false) {
                outWriter.write(mobile);
                outWriter.newLine();
            }
        }
        sendReader.close();

        outWriter.flush();
        outWriter.close();

        long end = System.currentTimeMillis();
        System.out.println(String.format("exec success! used %d ms", end - start));
    }
}
```


## 附录2 Constant pool
```
Constant pool:
    #1 = Methodref          #36.#75       // java/lang/Object."<init>":()V
    #2 = Fieldref           #23.#76       // me/fangfeng/filter/Trie.MAX_NODE:I
    #3 = Fieldref           #23.#77       // me/fangfeng/filter/Trie.CHAR_NUM:I
    #4 = Class              #49           // "[[I"
    #5 = Fieldref           #23.#78       // me/fangfeng/filter/Trie.nxt:[[I
    #6 = Fieldref           #23.#79       // me/fangfeng/filter/Trie.flag:[Z
    #7 = Fieldref           #23.#80       // me/fangfeng/filter/Trie.trieIndex:I
    #8 = Long               10l
   #10 = Methodref          #81.#82       // java/lang/System.currentTimeMillis:()J
   #11 = Class              #83           // java/io/BufferedReader
   #12 = Class              #84           // java/io/InputStreamReader
   #13 = Class              #85           // java/io/FileInputStream
   #14 = Methodref          #13.#86       // java/io/FileInputStream."<init>":(Ljava/lang/String;)V
   #15 = Methodref          #12.#87       // java/io/InputStreamReader."<init>":(Ljava/io/InputStream;)V
   #16 = Methodref          #11.#88       // java/io/BufferedReader."<init>":(Ljava/io/Reader;)V
   #17 = Class              #89           // java/io/BufferedWriter
   #18 = Class              #90           // java/io/OutputStreamWriter
   #19 = Class              #91           // java/io/FileOutputStream
   #20 = Methodref          #19.#86       // java/io/FileOutputStream."<init>":(Ljava/lang/String;)V
   #21 = Methodref          #18.#92       // java/io/OutputStreamWriter."<init>":(Ljava/io/OutputStream;)V
   #22 = Methodref          #17.#93       // java/io/BufferedWriter."<init>":(Ljava/io/Writer;)V
   #23 = Class              #94           // me/fangfeng/filter/Trie
   #24 = Methodref          #23.#75       // me/fangfeng/filter/Trie."<init>":()V
   #25 = Methodref          #11.#95       // java/io/BufferedReader.readLine:()Ljava/lang/String;
   #26 = Methodref          #96.#97       // java/lang/Long.parseLong:(Ljava/lang/String;)J
   #27 = Methodref          #23.#98       // me/fangfeng/filter/Trie.insert:(J)V
   #28 = Methodref          #11.#99       // java/io/BufferedReader.close:()V
   #29 = Methodref          #23.#100      // me/fangfeng/filter/Trie.query:(J)Z
   #30 = Methodref          #17.#101      // java/io/BufferedWriter.write:(Ljava/lang/String;)V
   #31 = Methodref          #17.#102      // java/io/BufferedWriter.newLine:()V
   #32 = Methodref          #17.#103      // java/io/BufferedWriter.flush:()V
   #33 = Methodref          #17.#99       // java/io/BufferedWriter.close:()V
   #34 = Fieldref           #81.#104      // java/lang/System.out:Ljava/io/PrintStream;
   #35 = String             #105          // exec success! used %d ms
   #36 = Class              #106          // java/lang/Object
   #37 = Methodref          #96.#107      // java/lang/Long.valueOf:(J)Ljava/lang/Long;
   #38 = Methodref          #108.#109     // java/lang/String.format:(Ljava/lang/String;[Ljava/lang/Object;)Ljava/lang/String;
   #39 = Methodref          #110.#111     // java/io/PrintStream.println:(Ljava/lang/String;)V
   #40 = Integer            700000
   #41 = Fieldref           #23.#112      // me/fangfeng/filter/Trie.MAX_ITEM:I
   #42 = Fieldref           #23.#113      // me/fangfeng/filter/Trie.AVG_LENGTH:I
   #43 = Utf8               MAX_ITEM
   #44 = Utf8               I
   #45 = Utf8               AVG_LENGTH
   #46 = Utf8               MAX_NODE
   #47 = Utf8               CHAR_NUM
   #48 = Utf8               nxt
   #49 = Utf8               [[I
   #50 = Utf8               flag
   #51 = Utf8               [Z
   #52 = Utf8               trieIndex
   #53 = Utf8               <init>
   #54 = Utf8               ()V
   #55 = Utf8               Code
   #56 = Utf8               LineNumberTable
   #57 = Utf8               insert
   #58 = Utf8               (J)V
   #59 = Utf8               StackMapTable
   #60 = Utf8               query
   #61 = Utf8               (J)Z
   #62 = Utf8               main
   #63 = Utf8               ([Ljava/lang/String;)V
   #64 = Class              #114          // "[Ljava/lang/String;"
   #65 = Class              #115          // java/lang/String
   #66 = Class              #83           // java/io/BufferedReader
   #67 = Class              #89           // java/io/BufferedWriter
   #68 = Class              #94           // me/fangfeng/filter/Trie
   #69 = Utf8               Exceptions
   #70 = Class              #116          // java/io/FileNotFoundException
   #71 = Class              #117          // java/io/IOException
   #72 = Utf8               <clinit>
   #73 = Utf8               SourceFile
   #74 = Utf8               Trie.java
   #75 = NameAndType        #53:#54       // "<init>":()V
   #76 = NameAndType        #46:#44       // MAX_NODE:I
   #77 = NameAndType        #47:#44       // CHAR_NUM:I
   #78 = NameAndType        #48:#49       // nxt:[[I
   #79 = NameAndType        #50:#51       // flag:[Z
   #80 = NameAndType        #52:#44       // trieIndex:I
   #81 = Class              #118          // java/lang/System
   #82 = NameAndType        #119:#120     // currentTimeMillis:()J
   #83 = Utf8               java/io/BufferedReader
   #84 = Utf8               java/io/InputStreamReader
   #85 = Utf8               java/io/FileInputStream
   #86 = NameAndType        #53:#121      // "<init>":(Ljava/lang/String;)V
   #87 = NameAndType        #53:#122      // "<init>":(Ljava/io/InputStream;)V
   #88 = NameAndType        #53:#123      // "<init>":(Ljava/io/Reader;)V
   #89 = Utf8               java/io/BufferedWriter
   #90 = Utf8               java/io/OutputStreamWriter
   #91 = Utf8               java/io/FileOutputStream
   #92 = NameAndType        #53:#124      // "<init>":(Ljava/io/OutputStream;)V
   #93 = NameAndType        #53:#125      // "<init>":(Ljava/io/Writer;)V
   #94 = Utf8               me/fangfeng/filter/Trie
   #95 = NameAndType        #126:#127     // readLine:()Ljava/lang/String;
   #96 = Class              #128          // java/lang/Long
   #97 = NameAndType        #129:#130     // parseLong:(Ljava/lang/String;)J
   #98 = NameAndType        #57:#58       // insert:(J)V
   #99 = NameAndType        #131:#54      // close:()V
  #100 = NameAndType        #60:#61       // query:(J)Z
  #101 = NameAndType        #132:#121     // write:(Ljava/lang/String;)V
  #102 = NameAndType        #133:#54      // newLine:()V
  #103 = NameAndType        #134:#54      // flush:()V
  #104 = NameAndType        #135:#136     // out:Ljava/io/PrintStream;
  #105 = Utf8               exec success! used %d ms
  #106 = Utf8               java/lang/Object
  #107 = NameAndType        #137:#138     // valueOf:(J)Ljava/lang/Long;
  #108 = Class              #115          // java/lang/String
  #109 = NameAndType        #139:#140     // format:(Ljava/lang/String;[Ljava/lang/Object;)Ljava/lang/String;
  #110 = Class              #141          // java/io/PrintStream
  #111 = NameAndType        #142:#121     // println:(Ljava/lang/String;)V
  #112 = NameAndType        #43:#44       // MAX_ITEM:I
  #113 = NameAndType        #45:#44       // AVG_LENGTH:I
  #114 = Utf8               [Ljava/lang/String;
  #115 = Utf8               java/lang/String
  #116 = Utf8               java/io/FileNotFoundException
  #117 = Utf8               java/io/IOException
  #118 = Utf8               java/lang/System
  #119 = Utf8               currentTimeMillis
  #120 = Utf8               ()J
  #121 = Utf8               (Ljava/lang/String;)V
  #122 = Utf8               (Ljava/io/InputStream;)V
  #123 = Utf8               (Ljava/io/Reader;)V
  #124 = Utf8               (Ljava/io/OutputStream;)V
  #125 = Utf8               (Ljava/io/Writer;)V
  #126 = Utf8               readLine
  #127 = Utf8               ()Ljava/lang/String;
  #128 = Utf8               java/lang/Long
  #129 = Utf8               parseLong
  #130 = Utf8               (Ljava/lang/String;)J
  #131 = Utf8               close
  #132 = Utf8               write
  #133 = Utf8               newLine
  #134 = Utf8               flush
  #135 = Utf8               out
  #136 = Utf8               Ljava/io/PrintStream;
  #137 = Utf8               valueOf
  #138 = Utf8               (J)Ljava/lang/Long;
  #139 = Utf8               format
  #140 = Utf8               (Ljava/lang/String;[Ljava/lang/Object;)Ljava/lang/String;
  #141 = Utf8               java/io/PrintStream
  #142 = Utf8               println
```

## 参考

\[1\]: https://docs.oracle.com/javase/specs/jvms/se8/html/index.html "Java Virtual Machine Specification"
\[2\]: https://icyfenix.iteye.com/blog/1256329 "Java虚拟机规范（Java SE 7 中文版）"
```plain
  __                    __                  
 / _| __ _ _ __   __ _ / _| ___ _ __   __ _ 
| |_ / _` | '_ \ / _` | |_ / _ \ '_ \ / _` |
|  _| (_| | | | | (_| |  _|  __/ | | | (_| |
|_|  \__,_|_| |_|\__, |_|  \___|_| |_|\__, |
                 |___/                |___/ 
```

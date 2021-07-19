---
title: JVM 指令简析
author: fangfeng
date: 2018-07-24
categories:
  - 技术
tags:
  - JVM
  - Instruction
---

在之前描述过包括 ASM, CGlib, Java Proxy 的基本内容之后，本文将就更为基础的 JVM 指令集进行简单而有效的介绍。

当然，在开始正文前，读者需要了解到，JVM 指令集这种类似于汇编的规范性内容，包含一百多个指令，若要求一一介绍。
那么，直接阅读 [官方文档](https://docs.oracle.com/javase/specs/jvms/se8/html/index.html) 绝对是比本文的内容更为详实且准确。

这篇文档的目的，只是为了使读者建立起关于 JVM 指令集基本的常识性观念。

<!--more-->

## 术语约定

首先，需要就 `术语` 进行一些基础性的约定:

### 变量

![变量](https://ws3.sinaimg.cn/large/006tNc79gy1ftidi0xpmij30oa07kaaq.jpg)

- `变量`: 在类中，区分于 `方法` 的声明
  - `成员变量`: 作用域为整个类，在方法体与语句块之外声明的内容。在 `字节码` 中通常被称为 `字段(Field)`
    - `类成员变量 / 静态成员变量`: 被 `static` 修饰的 `成员变量`。一个类只有一份，在类被加载的时候即初始化。
    - `实例成员变量`: 非 `static` 修饰的 `成员变量`。随着类被实例化而进行初始化，每个实例对象都有一份特有的 `实例变量`。
  - `局部变量`: 作用域为方法体或者语句块。

### JVM 指令

通常，我们借助于 `javap` 命令来对 .class 文件的字节码内容进行查阅。

类似于汇编代码，`javap` 打印的JVM 指令将以下列格式进行展示:

```plain
<index> <opcode> [<operand1> [<operand2> ...]] [<comment>]
```

其中 
- `<index>` 指在 `code[]` 属性中这条指令的偏移量(从 0 开始计数)。
- `<opcode>` 指 `操作码`
- `<operandX>` 指 `操作数`，每个 `<opcode>` 都需要确定数量的操作数(规范中已经确定)。
- `<comment>` 指注释

## 指令集概览

首先，Java 代码经编译后的所有指令都基于 `方法(Method)` 被定义在 `Code` 属性中。

在 [ClassFile](https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-4.html#jvms-4.7.3) 的 `Code` 属性，结构定义如下:

```java
Code_attribute {
    // 其中 u1, u2, u4 分别表示这个变量所占的字节长度
    u2 attribute_name_index;                    // 属性名在常量池中的 index (执行常量池中 Code 的位置)
    u4 attribute_length;                        // 属性长度，不包括开始的六个字节
    u2 max_stack;                               // 运行时操作数栈的最大深度
    u2 max_locals;                              // 运行时所需的局部变量表的大小
    u4 code_length;                             // code 数组的长度
    u1 code[code_length];                       // code 数组，编译后方法体的内容都通过字节码指令存储在这里
    u2 exception_table_length;                  // 异常表的长度
    {   u2 start_pc;
        u2 end_pc;
        u2 handler_pc;
        u2 catch_type;
    } exception_table[exception_table_length]; // 异常表，Java 代码中所有的 catch, finally 的捕获都将由此表进行实现
    u2 attributes_count;                       // 属性计数
    attribute_info attributes[attributes_count];
}
```

Java 虚拟机的指令是由一个字节长度的 `操作码` 配合上其后的 0 个或多个 `操作数` 所构成的。

其中，`操作数` 的数量取决于 `操作码`，不同的 `操作码` 需要不同数量的 `操作数`。

按照类型划分，`操作数` 主要包括下列几类:

- 加载与存储指令，例如 iload, istore 等
- 运算指令，例如 iadd, isub, imul 等
- 类型转换指令，例如 i2b, i2s 等
- 对象创建与操作指令，例如 new, newarray 等
- 操作数栈管理指令，例如 dup, pop 等
- 控制转移指令，例如 if\_icmpeq 等
- 方法调用与返回指令，例如 invokevirtual, invokestatic 等
- 抛出异常指令，例如 athrow 等
- 同步指令，例如 monitorenter 等

**举几个简单的例子:**

`iadd` 指令，表示对两个 int 数的相加指令。将从操作数栈顶依次取出两个数，并相加，再重新压入操作数栈中。

`bipush 100` ，其中 `bipush` 是指令，后随一个操作数，表示把 `操作数 100 这个 byte 类型的数` 压入操作数栈顶

## 运行时数据区

JVM 定义了若干种运行期间会使用到的运行时数据区，见下图:

![JVM 运行时数据区](https://ws4.sinaimg.cn/large/006tNc79gy1ft5osi6it1j31kw0ql78e.jpg)

至于每一个的具体意义，在此不做详细展开，可用参考:

- 由周志明等翻译的 《Java 虚拟机规范(Java SE 7版) 》 2.5节内容 [链接](https://icyfenix.iteye.com/blog/1256329)
- [JVMS 2.5. Run-Time Data Areas](https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-2.html#jvms-2.5)

## Getter, Setter 的指令代码

首先，有必要提及，想要得到 .class 可视且友好的展示结果，可以使用 JDK 自带的 `javap` 命令。

本节要展示的内容是常见的 POJO 的 get, set 方法的指令代码。使用的示例代码如下:

```java
public class Test {
    
    private int number;

    public int getNumber() {
        return number;
    }

    public void setNumber(int number) {
        this.number = number;
    }
}
```

在经过 `javac` 编译，`javap` 解析之后，我们将看到下列内容

```plain
Classfile /Users/fangfeng/WorkPkg/lab/src/main/java/me/fangfeng/asm/Test.class
  Last modified Jul 23, 2018; size 357 bytes
  MD5 checksum bb1940cc6534d789359295b8dc80233b
  Compiled from "Test.java"
public class me.fangfeng.asm.Test
  minor version: 0
  major version: 52
  flags: ACC_PUBLIC, ACC_SUPER
Constant pool:
   #1 = Methodref          #4.#17         // java/lang/Object."<init>":()V
   #2 = Fieldref           #3.#18         // me/fangfeng/asm/Test.number:I
   #3 = Class              #19            // me/fangfeng/asm/Test
   #4 = Class              #20            // java/lang/Object
   #5 = Utf8               number
   #6 = Utf8               I
   #7 = Utf8               <init>
   #8 = Utf8               ()V
   #9 = Utf8               Code
  #10 = Utf8               LineNumberTable
  #11 = Utf8               getNumber
  #12 = Utf8               ()I
  #13 = Utf8               setNumber
  #14 = Utf8               (I)V
  #15 = Utf8               SourceFile
  #16 = Utf8               Test.java
  #17 = NameAndType        #7:#8          // "<init>":()V
  #18 = NameAndType        #5:#6          // number:I
  #19 = Utf8               me/fangfeng/asm/Test
  #20 = Utf8               java/lang/Object
{
  public me.fangfeng.asm.Test();
    descriptor: ()V
    flags: ACC_PUBLIC
    Code:
      stack=1, locals=1, args_size=1
         0: aload_0
         1: invokespecial #1                  // Method java/lang/Object."<init>":()V
         4: return
      LineNumberTable:
        line 3: 0

  public int getNumber();
    descriptor: ()I
    flags: ACC_PUBLIC
    Code:
      stack=1, locals=1, args_size=1
         0: aload_0
         1: getfield      #2                  // Field number:I
         4: ireturn
      LineNumberTable:
        line 8: 0

  public void setNumber(int);
    descriptor: (I)V
    flags: ACC_PUBLIC
    Code:
      stack=2, locals=2, args_size=2
         0: aload_0
         1: iload_1
         2: putfield      #2                  // Field number:I
         5: return
      LineNumberTable:
        line 12: 0
        line 13: 5
}
SourceFile: "Test.java"
```

打印出来的内容包括有 类的全限定名，访问控制权限，父类，接口，常量池以及各个方法。

以 `getNumber` 为例:

```plain
public int getNumber();
  descriptor: ()I
  flags: ACC_PUBLIC
  Code:
    stack=1, locals=1, args_size=1
       0: aload_0
       1: getfield      #2                  // Field number:I
       4: ireturn
    LineNumberTable:
      line 8: 0
```

- descriptor: 表示方法描述符，其中 `()` 内容表示入参，`I` 表示返回值的类型
- flags     : 表示方法的访问权限，当前限定为 `public`
- Code      : 存储有当前方法体指令码的一种方法内部属性。
    - stack : 表示当前方法(意即在运行时所处的栈帧，每个方法的调用都将在 `虚拟机栈` 中构建一个新的 `栈帧`) 使用的 `操作数栈的最大深度`
    - locals: 表示当前方法使用的 `局部变量表` 的大小
    - args\_size : 表示变量个数
    - LineNumberTable : 与 Debug 有关，指当前的的指令集在源文件中的第几行开始

```plain
0: aload_0              // 从局部变量表中加载 0 号元素到操作数栈中
1: getfield #2          // 获取常量池中 2 号元素(即 Field number:I) 的值，并加载到操作数栈中
4: ireturn              // 抛出当前操作数栈顶元素作为返回值
```

其中，每条指令前的 0, 1, 4 指当前指令作为 `Code` 属性的内容的偏移量。

换一句话说，`aload_0` 是 Code 属性 code[] 的第 0 个字节的内容
`getfield #2` 的是从 code[] 的第 1 个字节开始的。
`ireturn` 是从 code[] 的第 4 个字节开始。

至于为什么每条指令的开始位置不同，这取决于每条指令的长度。`aload_0` 指令本身为 1 字节的长度，且不要求附带操作数。
`getfield` 指令自长 1 个字节，但需要一个长度为 2 字节的常量池索引。因此 `ireturn` 将从第 4 字节开始

--- 

同时，可能有人会有所疑问，`aload_0` 加载的 0 号元素是什么？它貌似没有被用到？

首先，在每个方法被触发，在构建新的栈帧时，`this` 将自动作为 0 号元素被存入新栈帧的局部变量表作为 0 号元素。
同时，如果方法有入参，则入参按入参声明顺序依次作为 1, 2, 3, ... 元素存入。
(当然，这里存在特殊情况，诸如 long, double 这样长为 8 字节的元素，将占用 2 个局部变量表的数组下标，而其他元素顺延)。

至于看似 0 号元素 `this` 并没有被用到。事实上，它是作为 `getfield` 的一个限定被使用的。
试想，`getfield` 虽然通过 `#2` 能够知道需要获取到的变量名为 `number` 类型为 `I(即 int)` 的元素。但是，这个元素究竟属于哪个实例？
而操作数栈顶的 `this` 恰恰是指明，需要使用当前方法所在的类的 number 变量。

---

类似的，我们看一下 `setNumber` 方法

```plain
0: aload_0              // 从局部变量表中加载 0 号元素到操作数栈中 (用于确定字段所属的类)
1: iload_1              // 从局部变量表中加载 1 号元素到操作数栈中 (用于确定将要给字段赋的值)
2: putfield      #2     // 获取常量池中 2 号元素(即 Field number:I) 的地址，并赋值
5: return               // 无返回值的 return 指令来结束当前栈帧的执行
```

## 给变量赋初始值

经常会见到在方法体内部有类似这样的声明 `int score = 100` ，那么这样的内容翻译成指令会是如何？

对于较小的值，例如 100，将通过 `bipush 100`, `istore_1(假设用局部变量表 1 号元素存储 score 变量` 类似的形式进行赋值。
类似的，还是 `sipush` ，两者的区别是 bipush 可以支持 1 字节大小的整数，sipush 支持 2 字节大小的整数。

但是，int 最大可是可以到达 4 字节，更甚者，long 将达到 8 字节。

这时候，将要借助的就是 `ldc #<index>` 用来读取常量池编号小于 128 的常量(例如整数常量，浮点常量等等)
那么，超出 128 编号的？使用 `ldc_w #<index>` 读取 2 字节内容作为编号，最大 65535。当然，更大的内容，常量池都不能支持了哈。

同时，区别于将 byte, short, int, float 统一作为 4 字节结果读取，对于 long, double 这样的八字节元素，使用 `ldc2_w`

## 控制结构

作为一门图灵完备的语言，至少，控制结构是必不可少的元素。

那么，类似 `for(int i=0;i<10;i++)` 的 Java 代码编译成指令到底是什么样的呢？

```plain
0: iconst_0                     // 赋值指令，往操作数栈顶添加整数值 0
1: istore_1                     // 将操作数栈顶的元素存入局部变量表 1 号位置
2: iload_1                      // 从局部变量表 1 号位置加载元素到操作数栈
3: bipush        10             // 往操作数栈顶压入 byte 型值 10
5: if_icmpge     21             // 将操作数栈顶的两个元素进行比较, 如果 次顶部元素 >= 顶部元素，则重定向到偏移量为 21 的指令
//  for (...) {} 语句块的内容
15: iinc          1, 1          // 将局部变量表的 1 号位置元素 +1
18: goto          2             // 跳转到偏移量为 2 的指令
21: return                      // 调用无返回值的 return
```

类似的，`if(...)` 语句的比较较之 `for(;;)` 就更为简单。类比偏移量为 5 的指令即可。

## 调用方法

JVM 指令集中总计有 4 种调用方法的指令，包括有: 

- `invokevirtual`, 对普通实例方法的调用，将根据对象类型进行分发调用
- `invokestatic`, 对静态方法的调用
- `invokespecial`, 用于调用类的初始化方法，也用于调用父类方法和私有方法
- `invokeinterface`, 用于调用接口方法

以执行 `System.out.println()` 为例
假设常量池内容存在目标元素(具体以相应注释为准)

```plain
getstatic     #3                  // Field java/lang/System.out:Ljava/io/PrintStream;
invokevirtual #4                  // Method java/io/PrintStream.println:()V
```

## More 

更多内容，包括: 如何实例化一个对象，如何抛出及处理异常，代码同步声明等，暂且不表。

有时间再做补充

```plain
  __                    __                  
 / _| __ _ _ __   __ _ / _| ___ _ __   __ _ 
| |_ / _` | '_ \ / _` | |_ / _ \ '_ \ / _` |
|  _| (_| | | | | (_| |  _|  __/ | | | (_| |
|_|  \__,_|_| |_|\__, |_|  \___|_| |_|\__, |
                 |___/                |___/ 
```

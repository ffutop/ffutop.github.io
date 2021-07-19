---
title: Preprocessor Output
author: fangfeng
date: 2018-09-08
categories:
  - 技术
tags:
  - C
  - CPP
  - GCC
---

最近重新开始回顾 C 语言以及其编译后的文件格式 ELF。
暂时告别一步到位的命令 `gcc main.c`，如果从 `.c` 文件的编译来说，主要分为预编译(preprocess)、编译(Compilation)、汇编(Assembly)、链接(Linking) 四个步骤。
但是，仅仅从第一个流程 **预编译** 而言，就已经遇到了一些麻烦。

<small>program.i</small>
```c
# 1 "program.c"
# 1 "<built-in>"
# 1 "<command-line>"
# 1 "/usr/include/stdc-predef.h" 1 3 4
# 1 "<command-line>" 2
# 1 "program.c"
# 1 "header.h" 1
char *test(void);
# 2 "program.c" 2

int main(void)
{
}
```

预编译后的问题出现了诸如 `# 1 "program.c"` 的 *注释?* 

这里简单记录预处理输出文件的基本格式，方便今后回顾。
<!--more-->

## Output File format

首先，从预编译的结果看，`cpp (C preprocessor)` 程序主要是处理了所有的**宏指令**。然后添加上了一些所谓**线性标记**的内容。
最终得到的就是类似 `program.i` 的结果。

从细节上来说: 
首先，所有的宏指令，包括 `#include` (用于引入用户自定义及系统预定义的头文件)、`#define` (用于将使用到的宏进行替换)。
当然，在替换完成后，这些指令所在的行将被替换成空行。同时，所有的注释行也将被替换成空行。

此外，将添加上注入 `#1 "program.c"` 的**线性标记**。

### 线性标记

线性标记的标准格式:

`# linenum filename flags`

linenum 是为了配合预定义宏 `__LINE__` 是使用的，用于定位紧随的下一行内容在原文件中所在的行。

filename 指出了接下来的内容来自哪个原文件

flags 有如下几个取值:

- 1 : 表示这是一个新的文件的开始
- 2 : 表示回到文件 `filename` 的内容 (从其他的 *include* 的文件中)
- 3 : 表示接下来的内容是从系统预定义头文件中来的，因此需要抑制某些警告
- 4 : 表示接下来的内容需要被视作是被封装的隐式 `extern "C"` 块

### 实例解读

<small>program.c</small>
```c
#include "header.h"
int main(void)
{
    puts(test());
}
```

<small>header.h</small>
```c
char *test(void);
```

接下来的演示都将以 **program.c** 和 **header.h** 两个文件作为标准示例。
期间，将对 **program.c** or **header.h** 做不同程度的修改，已达到更好的展示效果。
*注意:* 额外添加的注释由 `!` 开始到该行结束(并不符合 C 语言语法)，但是帮助理解

**Sample 1**

直接利用 program.c 与 header.h 进行预编译 `cpp -o program.i program.c`
结果如下:

```c
# 1 "program.c"
# 1 "<built-in>"
# 1 "<command-line>"
# 1 "/usr/include/stdc-predef.h" 1 3 4
# 1 "<command-line>" 2
# 1 "program.c"
# 1 "header.h" 1            ! 下一行代码对应的是 header.h 文件第一行
char *test(void);
# 2 "program.c" 2           ! flag=2 表示下列内容由回到了 program.c 中，下一行对应原文件第二行

int main(void)
{
    puts(test());
}
```

**Sample 2**

接着，给 **program.c** 加点注释，在加点空行

<small>program.c - sub 1<small>
```c
// This is comment          ! 这里添加了一行注释
#include "header.h"

                            ! 这里加了个空行
int main(void)
{
    puts(test());
}
```

再看看效果

```c
# 1 "program.c"
# 1 "<built-in>"
# 1 "<command-line>"
# 1 "/usr/include/stdc-predef.h" 1 3 4
# 1 "<command-line>" 2
# 1 "program.c"             ! 下一行代码对应原代码中第 1 行，本来是注释，但会被预编译器处理成空行 (多行将逐一处理，但行数不会变)

# 1 "header.h" 1            ! 下面将描述 header.h 的代码
char *test(void);
# 3 "program.c" 2           ! 继续描述 program.c ，下一行对应原代码第 3 行。至于第 2 行，就是 #include "header.h" ，不会直接表现了


int main(void)
{
    puts(test());
}
```

**Sample 3**

在改变一些

<small>program.c - sub 2</small>
```c
// This is comment
#include "header.h"
#define TEN 10

int main(void)
{
    puts(test());
}
```

看看结果

```c
# 1 "program.c"
# 1 "<built-in>"
# 1 "<command-line>"
# 1 "/usr/include/stdc-predef.h" 1 3 4
# 1 "<command-line>" 2
# 1 "program.c"

# 1 "header.h" 1
char *test(void);
# 3 "program.c" 2
                        ! 原代码中对应行是宏 #define TEN 10 ，已经被空行替换掉了。

int main(void)
{
    puts(test());
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

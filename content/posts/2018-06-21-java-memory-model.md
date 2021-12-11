---
title: java-memory-model 
date: 2018-06-21
author: fangfeng
categories:
  - 技术
tags:
  - Java
  - JVM
  - Memory Model
---

## JVM 运行时数据区

![](https://ws4.sinaimg.cn/large/006tNc79gy1ft5osi6it1j31kw0ql78e.jpg)
<!--more-->

### PC寄存器

与操作系统中的PC寄存器功能基本一致。毕竟 JVM 建立的初衷就是模拟一台机器。
指向存储在**方法区**的字节码methods\_info部分的内存地址。

### 虚拟机栈

**虚拟机栈**用于存储**栈帧**

### 本地方法栈

用于支持 native 方法的执行

### 栈帧

存储在**虚拟机栈**中，主要包括**局部变量表**和**操作数栈**(又称**当前栈帧的操作数栈**)以及**运行时常量池的引用**。

*仍然有必要区别两个概念: 操作数 & 指令*
*指令指使操作数进行相关操作的基本命令*
*操作数通常指整数、浮点数以及类型引用等*

### 方法区

**方法区**是多个线程共享的一块区域，主要用于存储I/O操作中读入的类/接口的字节码(ClassFile 文件)
包括有 constant\_pool, field\_info, method\_info, attribute\_info 等

### Java堆

用于存储各种类的实例对象
```plain
  __                    __                  
 / _| __ _ _ __   __ _ / _| ___ _ __   __ _ 
| |_ / _` | '_ \ / _` | |_ / _ \ '_ \ / _` |
|  _| (_| | | | | (_| |  _|  __/ | | | (_| |
|_|  \__,_|_| |_|\__, |_|  \___|_| |_|\__, |
                 |___/                |___/ 
```
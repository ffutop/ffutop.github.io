---
title: PNG 文件格式
author: fangfeng
date: 2019-05-10
categories:
  - 技术
tags:
  - File Format
  - PNG
---

PNG (Portable Network Graphics) 文件格式(第二版)。PNG 文件格式由两类结构——`PNG 签名(PNG Signature)`和若干`数据块(chunk)`组成。

PNG 签名相当于其他文件格式中的魔数，用于声明二进制数据所代表的格式。类似的有 `JAVA` class 文件的 `ca fe ba be`、`ELF` 文件的 `7f 45 4c 46` (.ELF) 。PNG 签名使用 `89 50 4e 47 0d 0a 1a 0a` (.PNG....) 作为魔数。

> HINT 上述的 `.` 仅仅是为了指代非打印字符，并非真的是 `点号`。

在 `PNG 签名` 数据之后，紧接着就是 `数据块(chunk)` 的数据。虽然统称`数据块`，但存在不同类型的数据块（例如 IHDR, PLTE, IDAT, IEND 等）。每个 PNG 文件可以有若干连续的数据块（至少 3 个数据块），其中第一个数据块和最后一个数据块类型分别是 IHDR、IEND 。

<!--more-->

## Chunk Layout  

数据块由三/四部分组成：长度、数据块类型、数据（可选）、CRC 校验码

```plain
+ ------ + ---------- + ---------- + --- +
| LENGTH | CHUNK TYPE | CHUNK DATA | CRC |
+ ------ + ---------- + ---------- + --- +
OR
+ --------- + ---------- + --- +
| LENGTH(0) | CHUNK TYPE | CRC |
+ --------- + ---------- + --- +
```

- 长度 (4 bytes, unsigned integer): 只负责记录数据(CHUNK DATA)的长度
- 数据块类型 (4 bytes, char sequence): 由 4 个大小写字母组成
- 数据: 一系列数据字符
- CRC (4 bytes): 循环冗余校验码。对数据块类型和数据两块二进制信息进行校验，不包括长度部分。

**数据类型**

有趣的是：由4个大小写字母组成的数据块类型还额外的携带了一些配置信息。通过大写代表0，小写代表1。这里提供了4 bits信息。
- 第一个字母: 
  - 0 (critical, 决定性的数据块)
  - 1 (ancillary, 辅助性的数据块)
- 第二个字母:
  - 0 (public, 由本协议预定义的数据块类型)
  - 1 (private, 自定义的数据块类型)
- 第三个字母:
  - 0 (遵循当前版本规范的数据块)
  - 1 (为未来版本预留的位)
- 第四个字母:
  - 0 (unsafe to copy)
  - 1 (safe to copy)

例如数据类型 `cHNk` 

```plain
cHNk  <-- 32 bit chunk type represented in text form
||||
|||+- Safe-to-copy bit is 1 (lower case letter; bit 5 is 1)
||+-- Reserved bit is 0     (upper case letter; bit 5 is 0)
|+--- Private bit is 0      (upper case letter; bit 5 is 0)
+---- Ancillary bit is 1    (lower case letter; bit 5 is 1)
```

<center>数据块顺序规范</center>
<center><small>Copied From [W3C PNG Specification (Table 5.3)](https://www.w3.org/TR/2003/REC-PNG-20031110/)</small></center>

| 决定性的数据块类型 |                  |                                                              |
| ------------------------------------------------------------ | ---------------- | ------------------------------------------------------------ |
| 类型                                                   | 是否允许出现多次 | 顺序限制                                        |
| [**IHDR**](https://www.w3.org/TR/2003/REC-PNG-20031110/#11IHDR) | No               | 必须出现在开始处                                               |
| [**PLTE**](https://www.w3.org/TR/2003/REC-PNG-20031110/#11PLTE) | No               | 必须早于第一个 **IDAT** |
| [**IDAT**](https://www.w3.org/TR/2003/REC-PNG-20031110/#11IDAT) | Yes              | 可选；多个 **IDAT** 必须连续，不允许间断 |
| [**IEND**](https://www.w3.org/TR/2003/REC-PNG-20031110/#11IEND) | No               | 必须出现在结束处                                              |
| 辅助性的数据块类型           |                  |                                                              |
| 类型                                                   | 是否允许出现多次 | 顺序限制                                        |
| [**cHRM**](https://www.w3.org/TR/2003/REC-PNG-20031110/#11cHRM) | No               | 可选；需早于 **PLTE** 和 **IDAT** 出现 |
| [**gAMA**](https://www.w3.org/TR/2003/REC-PNG-20031110/#11gAMA) | No               | 可选；需早于 **PLTE** 和 **IDAT** 出现 |
| [**iCCP**](https://www.w3.org/TR/2003/REC-PNG-20031110/#11iCCP) | No               | 可选；需早于 **PLTE** 和 **IDAT** 出现；与 **sBIT** 不能同时出现 |
| [**sBIT**](https://www.w3.org/TR/2003/REC-PNG-20031110/#11sBIT) | No               | 可选；需早于 **PLTE** 和 **IDAT** 出现 |
| [**sRGB**](https://www.w3.org/TR/2003/REC-PNG-20031110/#11sRGB) | No               | 可选；需早于 **PLTE** 和 **IDAT** 出现；与 **iCCP** 不能同时出现 |
| [**bKGD**](https://www.w3.org/TR/2003/REC-PNG-20031110/#11bKGD) | No               | 可选；在 **PLTE** 之后，**IDAT** 之前出现 |
| [**hIST**](https://www.w3.org/TR/2003/REC-PNG-20031110/#11hIST) | No               | 可选；在 **PLTE** 之后，**IDAT** 之前出现 |
| [**tRNS**](https://www.w3.org/TR/2003/REC-PNG-20031110/#11tRNS) | No               | 可选；在 **PLTE** 之后，**IDAT** 之前出现 |
| [**pHYs**](https://www.w3.org/TR/2003/REC-PNG-20031110/#11pHYs) | No               | 可选；在 **IDAT** 之前出现 |
| [**sPLT**](https://www.w3.org/TR/2003/REC-PNG-20031110/#11sPLT) | Yes              | 可选；在 **IDAT** 之前出现 |
| [**tIME**](https://www.w3.org/TR/2003/REC-PNG-20031110/#11tIME) | No               | 可选 |
| [**iTXt**](https://www.w3.org/TR/2003/REC-PNG-20031110/#11iTXt) | Yes              | 可选 |
| [**tEXt**](https://www.w3.org/TR/2003/REC-PNG-20031110/#11tEXt) | Yes              | 可选 |
| [**zTXt**](https://www.w3.org/TR/2003/REC-PNG-20031110/#11zTXt) | Yes              | 可选 |

如果想要可视化直观的了解，详见 [W3C PNG Specification (Figure 5.2/5.3)](https://www.w3.org/TR/2003/REC-PNG-20031110/#figure52)

## Color Type

根据图像的颜色类型划分，PNG 总计支持五类图像。

| PNG image type        | Colour type |
| --------------------- | ----------- |
| 灰度                  | 0           |
| 真彩色                | 2           |
| 索引颜色              | 3           |
| 有透明通道的灰度      | 4           |
| 有透明通道的真彩色    | 6           |

针对这五种类型，其实都支持透明度配置，只是各有区分罢了。灰度(0)、真彩色(2)、索引颜色(3) 通过 **tRNS** 数据块来维护透明度设置。其中灰度(0)、真彩色(2)支持由配置一个统一的透明度(2字节)，无法为每个像素单独配置透明度；索引颜色(3)支持为每个索引颜色设置透明度，但每个索引颜色的透明度设置最大只有 8 bits 的选择空间。

对于有透明通道的真彩色(6)、灰度(4)，它们不能拥有 **tRNS** 数据块，因为它们本身的就有透明通道来记录每个像素点的透明度。

## Interlacing and pass extraction

交错渲染，中文资料中更多看到“隔行渲染”的翻译。PNG 提供 Interlace 选项: 0(不交错渲染)、1(Adam7 算法交错渲染)。

Adam7 交错渲染可以简单的理解为将图像像素点矩阵划分为7级，如下表，逐级分别提取 1~7 级都能构成一个矩阵，每个级别与更高级别只是相近的像素点缺失了。但随着 1~7 级获得的像素点信息越多，图像就会越细腻。从而达到网络加载图片逐渐从模糊到清晰的效果。

```
1 6 4 6 2 6 4 6
7 7 7 7 7 7 7 7
5 6 5 6 5 6 5 6
7 7 7 7 7 7 7 7
3 6 4 6 3 6 4 6
7 7 7 7 7 7 7 7
5 6 5 6 5 6 5 6
7 7 7 7 7 7 7 7
```

## Chunk specifications

下列的各个数据块的内容，将以一套比较实际的二进制数据来辅助解析。

### IHDR

| | | 
| ------------------ | ------- |
| Width              | 4 bytes |
| Height             | 4 bytes |
| Bit depth          | 1 byte  |
| Colour type        | 1 byte  |
| Compression method | 1 byte  |
| Filter method      | 1 byte  |
| Interlace method   | 1 byte  |

```plain
0000 000d 4948 4452 0000 004c 0000 0019 
0806 0000 009b d4c8 6f

LENGTH: 0x0000000d
TYPE: 0x49484452 (IHDR)
Width: 0x0000004c (76)
Height: 0x00000019 (25)
Bit depth: 08 (每样本点取值区间 0 ~ 2^8-1)
Colour type: 06 (有透明通道的真彩色)
Compression method: 00 (默认)
Filter method: 00 (五种过滤算法自适应)
Interlace method: 00 (无交错渲染)
CRC: 0x9bd4c86f
```

### PLTE 

索引颜色(3)所使用的数据块，用于描述每个索引对应的 RGB 值（每个 RBG 值总计占 3 字节）

索引颜色主要见于使用较少颜色的图像中，对出现的颜色建立索引，每个像素点直接指向索引即可。

### IDAT

IDAT 数据块用于存储各像素点的数据，具体数据由 **IHDR** 中声明的 `Filter method` 和 `Compression method` 共同决定。

```plain
0000 0020 4944 4154 081d 63e8 64fd f9bf 
9de3 bb02 0308 8038 400a 8419 9840 0410 
3082 0800 dcfc 07b5 4c0c 837a 

LENGTH: 0x00000020 (32 bytes)
TYPE: 0x49444154 (IDAT)
DATA: 稍后解析
CRC: 0x4c0c837a
```

DATA 部分的数据由于已经被压缩过了，因此看不出任何与原图相关的内容 [原图](https://github.com/DorMOUSE-None/Repo/blob/master/z.png) 大小为 3×3 像素

OK，在 **IHDR** 有个数据描述的是使用的压缩算法 (Compression)，0 代表默认，即 PNG 规范所指出的 Zlib 。在 MacOS 下，抽取出 DATA 数据，用 `zlib_decompress` 解压，得到如下数据：

```plain
00000000: 0089 05f9 ff87 08f7 2000 0000 00  
0000000d: 0089 05f9 ff00 0000 ff00 0000 00  
0000001a: 0200 0000 0000 0000 0100 0000 00  
```

OK，现在应该能看出点内容来了。**Colour Type** 描述过图像是带透明通道的真彩色，每个样本点用 8 bits 来描述。3×3 的图像也就意味着需要 4 字节/像素 (RGBA) × 3 像素/行 × 3 行 = 36 字节。好吧，莫名其妙多了 3 字节？这当然与 `Filter method` 有关。用于描述图像每行的像素点将被如何描述。0 代表直接通过 RGBA 描述，而 2 代表需要借助上一行数据与当前行做累加。还有更多详见 [Filters](https://www.w3.org/TR/2003/REC-PNG-20031110/#9Filters)

每行像素点描述的最开始一个字节就是对 `Filter method` 所使用的类型的描述。

简单解析一下，9 个像素点的数据如下：

```plain
00000000: 8905f9ff 8708f720 00000000  
0000000d: 8905f9ff 000000ff 00000000  
0000001a: 8905f9ff 00000000 00000000  
```

去看看[实际的图像](https://github.com/DorMOUSE-None/Repo/blob/master/z.png)，恰如解析出的结果，完全匹配。

### IEND

**IEND** 的内容最为简单，只是为了标识 chunks 的结束。其内容就是 `00000000 49454e44 ae426082`

## More

PNG 规范的文档很长，对规范的描述也相当详细，除了对初次接触图像的非专业人士存在很大的障碍。有太多的新名词的出现。

本篇仅仅只是摘录及翻译了一部分内容，只是为了能够对 PNG 的文件格式建立初步的印象，以及能够简单解析一些图像信息。更多的内容请移步 [PNG 规范](https://www.w3.org/TR/2003/REC-PNG-20031110/)
```plain
  __                    __                  
 / _| __ _ _ __   __ _ / _| ___ _ __   __ _ 
| |_ / _` | '_ \ / _` | |_ / _ \ '_ \ / _` |
|  _| (_| | | | | (_| |  _|  __/ | | | (_| |
|_|  \__,_|_| |_|\__, |_|  \___|_| |_|\__, |
                 |___/                |___/ 
```

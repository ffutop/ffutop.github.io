---
title: "[译] FreeType 字形约定"
author: ffutop
date: 2024-06-19
categories:
  - 翻译
tags:
  - Font
  - Glyph
  - FreeType
---


## 译者序

本文来自 [FreeType Glyph Conventions](https://freetype.org/freetype2/docs/glyphs/index.html)，截取了第 I ~ V 节，第 VI、VII 节未做翻译。

主要介绍字体与字形数据文件是如何定义并渲染到输出设备。

译者水平有限，不免存在遗漏或错误之处。如有疑问，敬请查阅原文。

以下是译文。

## I. 排版的基本概念

### 1. 字体文件，格式与信息

字体(font)是各种字符图像的集合，可用于显示或打印文本。单字的图像具有一些共同的属性，包括外观、样式、衬线等。从排版角度来说，必须区分字体系列(font-family)及其多种字型(font-faces)，这些字体通常风格不同，但来自同一模板。

比如 Palatino Regular 和 Palatino Italic 是来自同一个字体系列的两种不同字型，该字体系列本身称为 Palatino。

术语“字体”在使用时往往带有模糊性，根据上下文指代给定的字体系列或给定的字型。例如，大多数文字处理软件用户使用“字体”来描述字体系列（例如，“Courier”、“Palatino”等）；但是，大多数字体系列都是通过多个数据文件提供的：对于 TrueType，通常每个字型都有一个数据文件（即arial.ttf 表示“Arial Regular”，ariali.ttf 表示“Arial Italic”，等等）。这些文件也被称为“字体”，但实际上指的是字型。

因此，数字字体(digital font)是一份数据文件，可能包含一个或多个字型。对于每个字型，它都包含字符图像、字符指标，以及对文本布局和特定字符编码处理很重要的其他类型的信息。在某些格式（如 Adobe’s Type 1）中，单个字型通过多个文件描述（一个文件包含字符图像，另一个文件包含字符度量）。我们将在本文档的大部分内容中忽略这些实现上的问题，并将数字字体视为单个文件，尽管 FreeType 2 能够正确支持多文件字体。

为方便起见，包含多种字型的文件称为字体集合。这种情况相当少见，但可以在许多亚洲字体中看到，这些字体包含给定脚本的两种或多种表示形式的图像（通常用于水平和垂直布局）。

### 2. 字符图像和映射

字符图像称为字形(glyphs)。取决于脚本、用法或上下文，单个字符可能有多个不同的图像，即多个字形。多个字符也可以合并采用单个字形（一个很好的例子是罗马连字，如“fi”和“fl”，它们可以用单个字形表示）。字符和字形之间的关系非常复杂，本文不会展开讨论。此外，不同字体或多或少采用各异的方案来存储和访问字形。为了简单起见，我们在使用 FreeType 时仅保留以下概念：

- 字体文件包含一组字形；每个字形都可以存储为位图、矢量表示或任何其他方案。这些字形可以以任何顺序存储在字体文件中，通常通过简单的字形索引进行访问。
- 字体文件包含一个或多个表，称为字符映射表(character maps)（也称为 charmap 或简称 cmap ），用于将给定编码（例如 ASCII、Unicode、Big5、ShiftJIS 等）的字符代码转换为相对于字体文件的字形索引。单个字体可能包含多个字符映射。例如，许多 OpenType 字体包含 Apple 特定的字符映射以及 Unicode 字符映射，这使得它们可以在 Mac 和 Windows 平台上使用。

### 3. 字符和字体规格

每个字形图像都与各种指标相关联，这些指标描述了如何放置和管理文本；有关详细信息，请参阅[**第 III 节**](#III.%20字形指标)。指标与字形位置、光标前进以及文本布局有关。它们对于在呈现文本字符串时计算文本流非常重要。

每种可缩放字体格式还包含一些全局指标，以标准单位表示，用于描述同一字面中所有字形的某些属性。全局指标包括字体的最大字形边界框、上升部(ascender)、下降部(descender)和文本高度等。

非可缩放字体格式也包含指标。但是，它们仅适用于一组给定的字符尺寸和分辨率，并且通常以像素表示。

## II. 字形轮廓

本节描述了 FreeType 以及客户端应用程序使用字形图像的可缩放表示方式（称为轮廓）。

### 1. 像素、点和设备分辨率

在处理计算机图形程序时，单个像素(pixel)的物理尺寸（无论是用于屏幕还是打印机）通常不是正方形，输出设备在水平和垂直方向上的分辨率(resolution)也都不同。

因此，通常通过以 dpi（每英寸像素点数）表示的两个数字来定义设备的特性。例如，分辨率为 300×600 dpi 的打印机在水平方向上每英寸有 300 个像素点，在垂直方向上每英寸有 600 个像素点。典型计算机显示器的分辨率随其尺寸而变化（10 英寸和 25 英寸显示器面向 1024×768 dpi 时的像素点大小显然不同），当然也随图形模式分辨率而变化。

因此，文本的大小通常以点(point)为单位，而不是设备特定的像素。点是一个物理单位，在数字排版中，1 点等于 1/72 英寸。例如，大多数使用拉丁文字的书籍的正文大小都在 10 到 14 点之间。

因此，可以用以下公式根据点的大小计算出文本的像素大小：

$$
\text{像素大小} = \text{点的大小} \times  \text{分辨率}\ /\ 72
$$

分辨率以 dpi 表示。由于水平和垂直分辨率可能不同，单个点大小通常定义不同的文本宽度和高度（以像素为单位）。

与通常的想法不同，“以像素为单位的文本大小”与字符在显示或打印时的实际尺寸没有直接关系。这两个概念之间的关系有点复杂，取决于字体设计师的一些设计选择。下一小节将对此进行更详细的描述（请参阅 EM 方块上的说明）。

### 2. 矢量表示

轮廓的源格式是一组称为轮廓线的闭合路径的集合。每段轮廓线界定了字形的外部或内部区域，可以由线段或贝塞尔弧组成。

其中弧线通过控制点定义，可以是二阶(conic Béziers)或三阶(cubic Béziers)贝塞尔曲线。因此，FreeType 需要为轮廓的每个点标识它的类型（标准点或控制点）。

每个字形的原始轮廓点都位于网格整点位置。这些点通常以 16 位整数网格坐标的形式存储在字体文件中，网格的原点位于 (0,0)，取值范围从 -32768 到 32767。（尽管点坐标在其他格式（例如 Type 1）中可以是浮点数，但为了简单起见，我们将分析限制为整数值。）

*网格总是像传统的数学二维平面一样，X 轴从左到右，Y 轴从下到上。*

在创建字形轮廓时，字体设计师使用一个称为 EM 方块的假想方块。通常，EM 方块可以被认为是绘制字符的平板。方块的大小（即其 XY 轴的网格单元数）非常重要，原因有二：

- 它是用于轮廓/字形缩放的基准大小。例如，12pt 在 300×300 dpi 对应于 12\*300/72 = 50 像素。如果直接渲染，这是 EM 方块在输出设备上显示的大小。换句话说，从网格单位缩放到像素使用以下公式：
    
    $$
    \text{像素大小} = \text{点的大小} \times {分辨率}\ /\ 72 \\
    \text{像素坐标} = \text{网格坐标} \times \text{像素大小}\ /\ \text{EM 大小}
    $$
    
    像素大小的另一个缩写是 **ppem**（pixels per EM, 每 EM 像素）；此值也可以是小数。请注意，并非所有地方都支持小数 ppem 值。
    
- EM 尺寸越大，设计人员在数字化设计轮廓时可以使用的分辨率就越大。例如，在 EM 尺寸为 4 个单位的极端示例中，EM 方块内只有 25 个点位置可用，这显然是不够的。典型的 TrueType 字体使用 2048 个单位的 EM 尺寸；Type 1 或 CFF PostScript 字体传统上使用 1000 个网格单位的 EM 尺寸（但点坐标可以表示为浮点值）。

请注意，如果字体设计师愿意，字形可以自由延伸到 EM 方块之外。因此，EM 方块只是传统排版中的一种惯例。

网格单元通常被称为字体单元或 EM 单元。

*如前所述，`pixel_size`上述公式中的计算值与屏幕上字符的大小没有直接关系。它只是显示 EM 方块的大小。每个字体设计师都可以自由地在方块内放置自己喜欢的字形。这解释了为什么以下文本的字母高度不一样，即使它们以相同的点大小显示：*

![https://freetype.org/freetype2/docs/glyphs/font-comparison.svg](https://img.ffutop.com/01076AC6-E258-4829-AFA8-E236D5136D38.svg)

可以看出，Courier 字体系列的字形比 Times New Roman 小，而 Times New Roman 本身又比 Arial 略小。

### 3. 提示和位图渲染

字体文件中存储的轮廓称为“主”轮廓，因为其点坐标以字体单元表示。在将其转换为位图之前，必须将其缩放到给定的大小和分辨率。这可以通过非常简单的转换来完成，但是在小尺寸时可能会出现不良伪像，特别是字母（如“E”或“H”）的主干宽度或高度不同的情况。

因此，正确的字形渲染需要将缩放的点沿目标设备像素网格对齐，这需要通过称为网格拟合的操作来实现。其主要目的之一是确保整个字体的宽度和高度得到忠实地渲染（例如，通常希望“I”和“T”字形的中心垂直线具有相同的像素宽度），以及管理诸如主干和悬垂之类的特征（这些特征可能会在小像素尺寸下发生问题）

有几种方法可以正确执行网格拟合；大多数可缩放格式将一些控制数据或程序与每个字形轮廓相关联。以下是概述：

- **显式网格拟合**
    
    TrueType 格式定义了一个基于堆栈的虚拟机，可以使用 200 多个运算符为其编写程序（也称为字节码），其中大多数与几何运算有关。因此，每个字形都由轮廓和控制程序组成，以按照字体设计师定义的方式执行实际的网格拟合。
    
- **隐式网格调整（也称为提示）**
    
    Type 1、CFF 和 CFF2 格式采用的方法要简单得多：每个字形由一个轮廓和几个称为提示的部分组成，这些提示用于描述字形的一些重要特征，例如是否存在主干、宽度是否规则等。提示类型并不多，最终渲染器将负责解释提示以生成合适的轮廓。
    
- **自动网格调整**
    
    有些格式除了字体度量（例如前进宽度和高度）之外，每个字形轮廓不包含任何控制信息。然后由渲染器“猜测”轮廓中更有趣的特征，以执行一些像样的网格拟合。
    

下表总结了每种方案的优缺点。

| 网格拟合方案 | 优点 | 缺点 |
| --- | --- | --- |
| 显式 | 质量。小尺寸也能获得出色的效果。这对于屏幕显示非常重要。
一致性。所有渲染器都会生成相同的字形位图（至少在理论上）。 | 速度。如果字形程序很复杂，则解释字节码可能会很慢。
大小。字形程序可以很长。
技术难度。编写好的提示程序极其困难。可用的工具非常少。 |
| 隐式 | 大小。提示通常比显式字形程序小得多。
速度。网格拟合通常是一个快速的过程。 | 质量。小尺寸时质量通常不理想。但使用抗锯齿功能效果会更好。
不一致。不同的渲染器，甚至同一引擎的不同版本，其结果可能会有所不同。 |
| 自适应 | 尺寸。无需控制信息，因此字体文件更小。
速度。取决于网格拟合算法。通常比显式网格拟合更快。 | 质量。小尺寸时质量通常不理想。但使用抗锯齿功能效果会更好。
速度。取决于网格拟合算法。
不一致。不同的渲染器，甚至同一引擎的不同版本，其结果可能会有所不同。 |

## III. 字形指标

### 1. 基线、笔和布局

基线是一条假想线，用于在呈现文本时引导字形。它可以是水平的（例如拉丁文、西里尔文、阿拉伯文）或垂直的（例如中文、日文、蒙古文）。此外，为了呈现文本，基线上有一个虚拟点，称为笔位置(pen position)或原点(origin)，用于定位字形。

每种布局都使用不同的字形放置约定：

- 在水平布局中，字形放置在基线上。通过移动笔的位置（向右或向左）来呈现文本。
    
    两个连续笔位置之间的距离是字形特定的，称为前进宽度(advance width)。请注意 ，即使对于从右到左的脚本（如阿拉伯语），其值始终为正数。这会导致文本呈现方式的一些差异。
    
    ![*笔的位置始终放在基线上*](https://img.ffutop.com/FEF53C38-A59C-414D-B647-CE509B3B7C60.svg)
    
    *笔的位置始终放在基线上*
    

在垂直布局中，字形以基线为中心：

![https://freetype.org/freetype2/docs/glyphs/glyph-metrics-2.svg](https://img.ffutop.com/8D69BF4E-C8C1-4A8C-90A3-7920E89478DC.svg)

### 2. 排版指标和边界框

给定字体的所有字型定义了各种字体规格。

- **上升线(Ascent)**
    
    从基线到用于放置轮廓点的最高或上部网格坐标的距离。由于网格的方向为Y 轴向上，因此它是一个正值。
    
- **下降线(Decent)**
    
    从基线到用于放置轮廓点的最低网格坐标的距离。在 FreeType 中，由于网格的方向，这是一个负值。请注意，在某些字体格式中，这是一个正值。
    
- **线隙(Linegap)**
    
    两行文本之间必须留出的距离。行间距（基线到基线的距离）应按以下方式计算：
    
    $$
    \text{行间距(linespace)} = \text{上升线}-\text{下降线} + \text{线隙}
    $$
    

其他的指标包括：

- **边界框(Bounding box)**
    
    这是一个虚构的框，通常尽可能紧密地包围字形。它由四个参数表示，即 xMin、yMin、 xMax 和 yMax，这些参数可以针对任何轮廓进行计算。如果在原始轮廓中测量，可以采用字体单位表示；如果在缩放轮廓中测量，可以采用整数（或分数）像素单位表示。
    
    边界框的常见简写是“bbox”。
    
- **内部行距**
    
    这个概念直接来自传统排版领域。它表示行距内为位于 EM 方块之外的字形特征（如重音）保留的空间量 。它通常可以计算为
    
    $$
    \text{内部行距} = \text{上升线} - \text{下降线} - \text{EM 大小}
    $$
    
- **外部行距**
    
    这是线隙的别名。
    

### 3. 字距和前进

每个字形还具有称为“字距”和 “前进”的距离。实际值取决于布局，因为同一个字形可用于水平或垂直呈现文本：

- **左侧字距**
    
    从当前笔位置到字形左边界框边缘的水平距离。对于水平布局，该距离为正，而对于垂直布局，大多数情况下为负。
    
    在 FreeType API 中，这也被称为 bearingX，或简写为 lsb。
    
- **顶侧字距**
    
    从基线到字形 bbox 顶部的垂直距离。对于水平布局，它通常为正数，对于垂直布局，它通常为负数。
    
    在 FreeType API 中，这也被称为 bearingY 。
    
- **前进宽度**
    
    处理文本时，在渲染字形后增加（对于从左到右书写）或减少（对于从右到左书写）笔位置的水平距离。对于水平布局，它始终为正，对于垂直布局，它始终为零。
    
    在 FreeType API 中，这也被称为 advanceX。
    
- **前进高度**
    
    渲染字形后，笔位置的垂直距离（对于从上到下的书写）或垂直距离（对于从下到上的书写，这种情况极为罕见）的减少量。对于水平布局，该值始终为零；对于垂直布局，该值始终为正值。
    
    在 FreeType API 中，这也被称为 advanceY。
    
- **字形宽度**
    
    字形的水平范围。对于未缩放的字体坐标，它是
    
    $$
    \text{字形宽度} = \text{bbox.xMax} - \text{bbox.xMin}
    $$
    
    对于缩放的字形，其计算需要特别注意，如下面的网格调整章节中所述。
    
- **字形高度**
    
    字形的垂直范围。对于未缩放的字体坐标，它是
    
    $$
    \text{字形高度} = \text{bbox.yMax} - \text{bbox.yMin}
    $$
    
    对于缩放的字形，其计算需要特别注意，如下面的网格调整章节中所述。
    
- **右侧字距**
    
    仅用于水平布局，描述从 bbox 的右边缘到前进宽度的距离。在大多数情况下它是一个非负数：
    
    $$
    \text{右侧字距} = \text{前进宽度} - \text{左侧字距} - (\text{xMax}-\text{xMin})
    $$
    
    该值的常见简写是 rsb。
    

下图给出了水平指标的所有细节：

![https://freetype.org/freetype2/docs/glyphs/glyph-metrics-3.svg](https://img.ffutop.com/E0F482A1-FE12-44DC-8855-2EAA09FE00E5.svg)

下面是垂直指标的所有细节：

![https://freetype.org/freetype2/docs/glyphs/glyph-metrics-4.svg](https://img.ffutop.com/B490E44F-31D4-466B-A3C2-B7BF7B63F06E.svg)

### 4. 网格拟合的效果

因为提示(hinting)将字形的控制点与像素网格对齐，所以这个过程会以不同于简单缩放的方式稍微修改字符图像的尺寸。

例如，小写字母“m”的图像有时适合主网格中的一个正方形。但是，为了使其在小像素尺寸下可读，提示往往会水平放大其轮廓，以使其三条腿清晰可见，从而产生更宽的字符位图。

字形指标也受到网格拟合过程的影响：

- 图像的宽度和高度发生了改变。即使只改变了一个像素，在小像素尺寸下也会产生很大的不同。
- 图像的边界框被修改，从而修改了字距。
- 必须更新前进量。例如，如果提示位图大于缩放位图，则必须增加前进宽度，以反映增强的字形宽度。

这有一些隐式处理：

- 由于提示，简单地缩放字体的上升线或下降线可能无法得到正确的结果。一种可行的解决方案是在缩放时保持上升线的上限、下降线的下限不变。
- 统一获取一批字形的提示字形和前进宽度是不现实的，因为提示在每个字体轮廓上的工作方式不同。唯一的解决方案是分别提示每个字形并记录返回的值（例如在缓存中）。某些格式（如 TrueType）甚至包含一个预先计算的一小组常见字符像素大小值的表。
- 提示取决于最终字符的宽度和高度（以像素为单位），这意味着提示强依赖于分辨率。这种特性使得正确的所见即所得布局难以实现。

使用 FreeType 对字形轮廓执行 2D 转换非常容易。但是，在对提示轮廓使用平移时，应始终注意**仅使用整数像素距离**（这意味着 API 函数的参数 FT_Outline_Translate 应全部为 64 的倍数，因为点坐标采用 26.6 定点格式）。否则，平移将毁掉提示器的工作，导致位图质量非常低！

### 5. 文本宽度和边界框

如前所述，给定字形的原点对应于笔在基线上的位置。与许多典型的位图字体格式不同，它不一定位于字形的边界框角之一上。在某些情况下，原点可能在边界框之外，在其他情况下，它可能在边界框之内，具体取决于给定字形的形状。

同样，字形的前进宽度是在布局期间应用于笔位置的增量，与字形的宽度无关，后者实际上是字形的边界框宽度。

相同的约定适用于文本字符串，并产生以下后果。

- 给定文本字符串的边界框不一定包含文本光标，文本光标也不位于其某个角上。
- 字符串的前进宽度与其边界框尺寸无关。特别是当字符串包含前导和尾随空格或制表符时。
- 最后，字距调整等额外处理会创建文本字符串，其尺寸与单个字形度量的简单并置没有直接关系。例如，“VA”的前进宽度不是“V”和“A”各自前进宽度的和。

## IV. 字距调整

术语“*字距调整”*是指用于调整文本字符串中连续字形的相对位置的特定信息。本节介绍几种类型的字距调整信息，以及在执行文本布局时处理它们的方式。

### 1. 字形对的字距调整

字距调整是指根据两个连续字形的轮廓来调整它们之间的间距。例如，可以轻松将 T 和 y 移近，因为 y 的顶部恰好位于 T 的右下方。

当只使用标准宽度来布局文本时，一些连续的字形看起来有点太近或太远。例如，以下单词中 A 和 V 之间的间距似乎比正常的要宽一些。

![https://freetype.org/freetype2/docs/glyphs/kerning-1.svg](https://img.ffutop.com/CC317B93-0372-4645-8041-1D787ACA6C93.svg)

如果两个字母字距略微减小：

![https://freetype.org/freetype2/docs/glyphs/kerning-2.svg](https://img.ffutop.com/195F304A-9B69-4973-9842-9BD9EAF72206.svg)

如您所见，这种调整可以带来很大的不同。因此，有些字体包含一个表格，用于描述若干给定字形对的字距调整。

- 这些字形对是有序的，即对 (A,V) 的字距调整不能应用于对 (V,A) 的字距调整。
- 取决于布局和/或脚本，字距调整可以是水平或垂直方向的，例如，一些水平布局（如阿拉伯语）可以利用连续字形之间的垂直字距调整。
- 字距调整以网格单位表示。它们通常沿 X 轴方向，这意味着负值表示两个字形在水平布局中必须设置得更近一些。

请注意，OpenType 字体 (OTF) 提供两种不同的字距调整机制，分别使用 kern 和 GPOS 表，它们是 OTF 文件的一部分。较旧的字体只包含前者，而较新的字体包含两个表或只包含 GPOS 表。FreeType 仅支持通过 kern 表进行字距调整。要解释 GPOS 表中的字距调整数据，您需要一个更高级的库，如 [**ICU**](http://icu-project.org/) 或 [**HarfBuzz**](http://harfbuzz.org/)，因为它可能依赖于上下文（例如，字距调整可能因文本字符串中的位置而异）。

### 2. 应用字距调整

在渲染文本时应用字距调整是一个相当简单的过程。它只是在渲染下一个字形之前将缩放后的字距距离添加到笔位置。但是，为了排版的正确，渲染器必须考虑更多细节。

滑动点(sliding dot)问题就是一个很好的例子：

许多字体在大写字母（如 T 或 F）和后面的点(.)之间都包含字距调整，以便将后者的字形恰好滑动到其字形主干右下角。

![https://freetype.org/freetype2/docs/glyphs/kerning-3.svg](https://img.ffutop.com/7D79F2C6-0BA3-4CE7-A83B-1482D5AB1810.svg)

但是，当点(.)之后没有空格存在时，上一个句子将变成

![https://freetype.org/freetype2/docs/glyphs/kerning-4.svg](https://img.ffutop.com/57A8232D-E448-4F8F-911D-37F5D34FD82D.svg)

这显然太过拥挤。如第一个示例所示，这里的解决方案是仅在条件符合时才滑动点。当然，这需要对文本的含义有一定的了解，而这正是 GPOS 字距调整的优势所在：根据上下文，可以应用不同的字距调整值来获得正确的排版结果。

## V. 文本处理

本节将介绍使用先前定义的概念来呈现文本的算法。当然，这里只介绍对拉丁文或西里尔文等简单文本处理方法。对于阿拉伯文或高棉文等需要字形整形操作的，不在本节的介绍范围（它应该有适当的布局引擎（如 [**Pango**](http://www.pango.org/)）来完成 ）。

### 1. 编写简单的文本字符串

在第一个示例中，我们将生成一串简单的拉丁文文本，即采用水平从左到右的布局。仅使用像素度量，该过程如下所示：

1. 将字符串转换为一系列字形索引。
2. 将笔放置到光标位置。
3. 获取或加载字形图像。
4. 平移字形，使得它的原点与笔的位置相匹配。
5. 将字形渲染到目标设备。
6. 按字形的前进宽度（以像素为单位）增加笔的位置。
7. 对于剩余的每个字形，从步骤 3 重新开始。
8. 当所有字形都完成后，将文本光标设置到新的笔位置。

请注意，字距调整不是该算法的一部分。

### 2. 子像素定位

此算法可用于无水平提示的提示模式。它本质上提供了所见即所得的文本布局。文本渲染与第 1 节中描述的算法非常相似，但有以下几点不同：

- 笔的位置以小数像素表示。
- 提示的轮廓必须水平移动到正确的子像素位置。
- 前进宽度以小数像素表示，不一定是整数。

以下是该算法的改进版本：

1. 将字符串转换为一系列字形索引。
2. 将笔放在光标位置。这可以是非整数点。
3. 获取或加载字形图像。
4. 通过 pen\_pos - floor(pen\_pos) 平移字形。
5. 将字形渲染到目标设备的 floor(pen\_pos) 处。
6. 将笔的位置按字形的前进宽度（以小数像素为单位）增加。
7. 对于剩余的每个字形，从步骤 3 重新开始。
8. 当所有字形都完成后，将文本光标设置到新的笔位置。

### 3. 简单的字距调整

将字距调整添加到基本文本渲染算法中很容易：当找到一个字形对的字距调整值时，只需在步骤 4 之前将缩放的字距调整值添加到笔位置即可。当然，在算法 1 的情况下，值应该四舍五入，但算法 2 则不需要。这给了我们：

带字距调整的算法 1：

1. 将字符串转换为一系列字形索引。
2. 将笔放置到光标位置。
3. 获取或加载字形图像。
4. 如果有的话，将四舍五入的字距调整值添加到笔位置。
5. 平移字形，使得它的原点与笔的位置相匹配。
6. 将字形渲染到目标设备。
7. 将笔的位置按字形的前进宽度（以像素为单位）增加。
8. 对于剩余的每个字形，从步骤 3 重新开始。

带字距调整的算法 2：

1. 将字符串转换为一系列字形索引。
2. 将笔放置到光标位置。
3. 获取或加载字形图像。
4. 如果有的话，将非四舍五入的字距调整值添加到笔位置。
5. 通过 pen\_pos - int(pen\_pos) 翻译字形。
6. 将字形渲染到目标设备的 int(pen\_pos) 处。
7. 将笔的位置按字形的前进宽度（以小数像素为单位）增加。
8. 对于剩余的每个字形，从步骤 3 重新开始。

### 4. 从右到左布局

从右到左布局的文本（如现代希伯来语文本）类似于从左到右的。唯一的区别是，在字形渲染之前必须减少笔位置（记住：前进宽度始终为正，即使对于希伯来语字形也是如此）。

从右到左算法1：

1. 将字符串转换为一系列字形索引。
2. 将笔放置到光标位置。
3. 获取或加载字形图像。
4. 将笔的位置减去字形的前进宽度（以像素为单位）。
5. 平移字形，使得它的原点与笔的位置相匹配。
6. 将字形渲染到目标设备。
7. 对于剩余的每个字形，从步骤 3 重新开始。

对算法 2 的更改以及字距调整的包含留给读者作为练习。

### 5. 垂直布局

布局垂直文本使用完全相同的流程，但有以下显著差异：

- 基线是垂直的，必须使用垂直度量而不是水平度量。
- 左侧字距通常为负，但这并不能改变字形原点必须位于基线上的事实。
- 前进高度始终为正，因此如果想要从上到下书写（假设 Y 轴向上），则必须减少笔的位置。

算法如下：

1. 将字符串转换为一系列字形索引。
2. 将笔放置到光标位置。
3. 获取或加载字形图像。
4. 平移字形，使得它的原点与笔的位置相匹配。
5. 将字形渲染到目标设备。
6. 将笔的垂直位置减少字形的前进高度（以像素为单位）。
7. 对于剩余的每个字形，从步骤 3 重新开始。
8. 当所有字形都完成后，将文本光标设置到新的笔位置。
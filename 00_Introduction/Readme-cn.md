# Part 0：介绍

> 粗浅的中文翻译版本，如有不当之处，请指出，感谢：）

我决定踏上编译器编写之旅。过去我写过一些[汇编器](https://github.com/DoctorWkt/pdp7-unix/blob/master/tools/as7)，也为无类型语言写过一个[简单编译器](https://github.com/DoctorWkt/h-compiler)。但是我从来没有写过可以自编译的编译器（a compiler that can compile itself）。这就是我此行的目的地。 

作为这个过程的一部分，我将把我的工作写下来，这样其他人就可以跟着做了。这也有助于我理清思路和想法。希望你和我会发现这很有用！

## 旅程的目标

以下是我这次旅行的目标和非目标（non-goals）：

- 写一个自编译的编译器（self-compiling compiler）。我认为如果编译器能够编译自己，它就可以称自己为“真正的”编译器。
- 瞄准至少一个真正的硬件平台。我见过一些为假想机器生成代码的编译器。我希望我的编译器能在真正的硬件上工作。还有，如果可能的话，我希望写的编译器可以支持不同硬件平台的多个后端。
- 实用先于研究。在编译器领域有很多研究。在这个旅程中，我想从零开始，所以我倾向于采用实用的方法，而不是偏重理论的方法。也就是说，有时候我需要引入（并实现）一些基于理论的东西。
- 遵循 KISS 原则：**keep it simple, stupid**！我肯定会在这里使用 Ken Thompson 的原则：“当有疑问时，使用暴力。”
- 采取许多小步骤来达到最终目标。我会把旅程分成许多简单的步骤，而不是大步跨越。这将使编译器的每一个新增部分都变得简单易懂。

## 目标语言

目标语言的选择是很困难的。如果我选择像 Python、Go 等这样的高级语言，那么我就必须实现一大堆库和类，因为它们是语言内置的。

我可以为 Lisp 这样的语言写一个编译器，但这可以很容易做到。

相反，我又回到了老本行，我打算为 C 语言的一个子集写一个编译器，足以让编译器自编译。

C 语言只是汇编语言的升级版（对于 C 语言的某个子集，而不是 C18），这使得将 C 代码编译成汇编的任务变得更加容易。哦，我也喜欢 C。

## 编译器工作的基础

编译器的工作是将一种语言（通常是高级语言）中的输入翻译成不同的输出语言（通常是比输入更低级别的语言）。主要步骤如下：

![](Figs/parsing_steps_cn.png)

- 通过[词法分析](https://en.wikipedia.org/wiki/Lexical_analysis)来识别词汇元素。在一些语言中，`=` 和 `==` 是不同的，所以不能只读一个 `=`。我们称这些词汇元素为标记（token）。

- [解析](https://en.wikipedia.org/wiki/Parsing)输入，即识别输入的语法和结构元素，并确保它们符合语言的语法。例如，你的语言可能有这样的决策结构：

  ```c
  if (x < 23) {
      print("x is smaller than 23\n");
  }
  ```

  但是在另一种语言中你可能会写成：

  ```c
  if (x < 23):
  	print("x is smaller than 23\n")
  ```

  这也是编译器可以检测语法错误的地方，比如第一个 print 语句末尾缺少分号。

- 对输入进行语义分析，即理解输入的含义。这实际上不同于识别语法和结构。例如，在英语中，句子的形式可能是`<主语><动词><形容词><宾语>`。以下两个句子结构相同，但含义完全不同：

  ```
  David ate lovely bananas.
  Jennifer hates green tomatoes.
  ```

- 将输入的含义翻译成不同的语言。在这里，我们将输入一部分一部分地转换为较低级别的语言。

## 资源

网上有很多编译器资源。下面是我会看的。

### 学习资源

如果你想从一些关于编译器的书籍、论文和工具开始，我强烈推荐以下列表：

- [Curated list of awesome resources on Compilers, Interpreters and Runtimes](https://github.com/aalhour/awesome-compilers) by Ahmad Alhour

### 现有编译器

当我要构建自己的编译器时，我计划看看其他编译器的想法，并可能会借用他们的一些代码。以下是我正在看的：

- [SubC](http://www.t3x.org/subc/) by Nils M Holm
- [Swieros C Compiler](https://github.com/rswier/swieros/blob/master/root/bin/c.c) by Robert Swierczek
- [fbcc](https://github.com/DoctorWkt/fbcc) by Fabrice Bellard
- [tcc](https://bellard.org/tcc/), also by Fabrice Bellard and others
- [catc](https://github.com/yui0/catc) by Yuichiro Nakada
- [amacc](https://github.com/jserv/amacc) by Jim Huang
- [Small C](https://en.wikipedia.org/wiki/Small-C) by Ron Cain, James E. Hendrix, derivatives by others

特别是，我将使用 SubC 编译器中的许多想法和一些代码。

### 设置开发环境

假设你想参加这次旅行，以下是你需要的。我将使用 Linux 开发环境，所以下载并设置你最喜欢的Linux 系统：我使用的是 Lubuntu 18.04。

 我将以两个硬件平台为目标：Intel x86-64 和 32 位 ARM。我将使用运行 Lubuntu 18.04 的 PC 作为 Intel 目标，使用运行 Raspbian 的树莓派作为 ARM 目标。

在 Intel 平台上，我们需要一个现有的 C 编译器。因此，安装这个软件包（我给出 Ubuntu/Debian 命令）：

```
$ sudo apt-get install build-essential
```

如果一个普通 Linux 系统需要更多的工具，请告诉我。
最后，克隆这个 Github 仓库。

## 下一步

在编译器编写旅程的下一部分中，我们将从扫描输入文件的代码开始，并查找作为语言词汇元素的标记。
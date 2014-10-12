## 调试器的工作原理第三章：调试信息
作者：Eli Bendersky &nbsp; 
译者：徐软件 &nbsp; [[原文链接](http://eli.thegreenplace.net/2011/02/07/how-debuggers-work-part-3-debugging-information/)]

本文是**调试器的工作原理**系列文章的第三章。阅读本文前，希望读者已经阅读过[第一章](/原理/2014/10/10/调试器的工作原理-第一章-基础/)和[第二章](/原理/2014/10/12/调试器的工作原理-第二章-断点)。

### 本文简介
本文将详细讨论调试过程中的两个问题：

* 如何从 C 语言代码中的变量和函数，映射到机器码的相应位置
* 映射过程中使用了哪些数据

### 调试信息
高级程序语言（如C/C++）有丰富的嵌套和迭代功能、自由的数据结构。现代编译器可以非常优雅的将这些高级程序语言转化成一大堆比特字节，即机器指令，目的是在特定 CPU 上跑的尽量快。C 语言中大部分代码行都将转化成多个机器指令。变量会出现在各个地方 —— 堆、栈、寄存器、甚至因优化而被删除。struct 和 object 并不显示的出现在机器指令中，而是内存中的地址偏移。

那么，当用户想跳进一个函数，调试器如何决定在哪个机器指令处设置中断指令？当用户想知道某个变量的值，调试器又如何能够找到相应的内存地址？答案是 —— 调试信息。

在编译器对高级语言执行编译的过程中，同时生成调试信息。调试信息是源代码与可执行文件之间的一种映射关系。这些信息是以一种预定义的格式跟机器指令保存在一起。这些预定义的格式，是根据不同平台的不同的可执行文件，经过几十年的经验累积形成的。本文当然不是介绍其发展历史，为了展示其运行过程，将选择一种格式详细展开。在现代的 Linux 系统和其他的 Unix-y 系统上的 ELF 可执行文件中，几乎是无处不在的一种格式：DWARF。

### DWARF 与 ELF
![DWARF debugging format](http://eli.thegreenplace.net/images/2011/02/dwarf_logo.gif)

> 有趣的是，ELF 指精灵，DWARF 指侏儒

ELF 是 Linux 系统上的主要可执行文件格式。根据 [wiki](http://en.wikipedia.org/wiki/DWARF) 上的介绍，DWARF 格式是专门为 ELF。理论上，它也可以服务于其他格式的二进制文件。

DWARF 格式复杂，它的诞生基于之前的多种系统和架构上的格式。复杂也是必然的，因为它解决了大难题 —— 给所有高级程序语言生成调试信息，支持所有平台和所有应用二进制接口（ABI）。DWARF 远远不是笔者的一篇拙劣的文章能解释清楚，可以说笔者也没有信心把它全部掌握。本文将结合例子，展示足够多的 DWARF 的实用信息，把调试信息的工作过程说清楚。

### ELF 文件的调试 section
首先粗看一下，ELF 文件中 DWARF 信息位于何处。ELF 文件可以包含任意的 section，一个 section header table 定义了文件中的 section 及其名字。不同工具需要不一样的 section ，可能链接器需要一部分，调试器需要另一部分。

本文使用 C 语言程序编译出的可执行文件做实验，编译出的文件名为 tracedprog2：

	#include <stdio.h>

	void do_stuff(int my_arg) {
		int my_local = my_arg + 2;
		int i;
		for (i = 0; i < my_local; ++i)
		printf("i = %d\n", i);
	}

	int main() {
		do_stuff(2);
		return 0;
	}

编译命令为：

	$ gcc -o tracedprog2.o -g -c tracedprog2.c

使用 `objdump -h` 命令可以把 ELF 文件中的头文件打印出来，可以看到有好几个名字以 `.debug_` 开头的 section，它们就是 DWARF 调试信息。

	Sections:
	Idx Name          Size      VMA       LMA       File off  Algn
	                  CONTENTS, ALLOC, LOAD, RELOC, READONLY, CODE
	  4 .debug_info   000000cd  00000000  00000000  00000094  2**0
	                  CONTENTS, RELOC, READONLY, DEBUGGING
	  5 .debug_abbrev 00000083  00000000  00000000  00000161  2**0
	                  CONTENTS, READONLY, DEBUGGING
	  6 .debug_loc    00000070  00000000  00000000  000001e4  2**0
	                  CONTENTS, READONLY, DEBUGGING
	  7 .debug_aranges 00000020  00000000  00000000  00000254  2**0
	                  CONTENTS, RELOC, READONLY, DEBUGGING
	  8 .debug_line   00000054  00000000  00000000  00000274  2**0
	                  CONTENTS, RELOC, READONLY, DEBUGGING
	  9 .debug_str    000000cd  00000000  00000000  000002c8  2**0
	                  CONTENTS, READONLY, DEBUGGING

每个 section 中第一个数字指长度，第四个数字指其起始点在 ELF 文件中的偏移，调试器使用这些信息读取文件中的 section 内容。

下面就根据一个实际的例子，来识别 DWARF 中有价值的调试信息。

### 寻找函数位置
调试中一个基本的操作是，跳进一个函数内部，所以需要在函数入口处设置断点。为了实现这一功能，在高级语言与底层机器指令之间，即函数名与函数入口的机器指令之间，需要有一张映射表。

该映射表可以再名为 `.debug_info` 的 section中找到。补充一个概念，DWARF 的基本实体叫做调试信息条目（Debugging Information Entry），简写为 DIE。每个 DIE 有个标签 - 描述它的类型，还有一组属性。DIE 是
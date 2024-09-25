---
title: 动态库相关问题
date: 2023-11-28 14:10:00 +0800
categories: [Blogging, compile]
tags: [writing]
---

动态库相较静态库的内容会稍微多一点，因为涉及到vm和relocation

## 翻译一下[Ref1-1](#ref)里的文章

文章重点讨论在32位x86架构上运行的Linux操作系统，但一般原理也适用于其他操作系统和处理器。值得注意的是，共享库有很多名称——共享库、共享对象、动态共享对象（DSOs）、动态链接库（DLL，如果您来自Windows背景）。为了保持一致性，我将在本文中尽量只使用“共享库”这个名称。

### 加载可执行文件

与其他支持虚拟内存的操作系统类似，Linux将可执行文件加载到固定的内存地址。如果我们检查某个随机可执行文件的ELF头部，我们会看到一个入口点地址：

```bash
$ readelf -h /usr/bin/uptime
ELF Header:
  Magic:   7f 45 4c 46 01 01 01 00 00 00 00 00 00 00 00 00
  Class:                             ELF32
  [...] some header fields
  Entry point address:               0x8048470
  [...] some header fields
```

这个地址是由链接器安排的，用于告诉操作系统从哪里开始执行可执行文件的代码。实际上，如果我们使用GDB加载可执行文件并检查地址`0x8048470`，我们会在那里看到可执行文件的.text段的第一条指令。

这意味着链接器在链接可执行文件时，可以完全解析所有内部符号引用（函数和数据）到固定且最终的位置。链接器会进行一些自身的重定位，但最终生成的输出不包含额外的重定位。

或者它确实需要呢？请注意，我在上一段中强调了“内部”这个词。只要可执行文件不需要共享库，它就不需要进行重定位。但是，如果它使用了共享库（就像大多数Linux应用程序一样），则需要对来自这些共享库的符号进行重定位，因为共享库的加载方式导致了这种需要。

### 导入共享库

与可执行文件不同，当构建共享库时，链接器无法假设其代码的已知加载地址。原因很简单。每个程序可以使用任意数量的共享库，无法提前知道任何给定共享库将加载到进程的虚拟内存的哪个位置。多年来，针对这个问题发明了许多解决方案，但在本文中，我将只关注Linux目前使用的解决方案。

但是首先，让我们简要地检查一下问题。这里是一些示例的C代码，我将其编译成一个共享库：

```cpp
int myglob = 42;

int ml_func(int a, int b)
{
    myglob += a;
    return b + myglob;
}
```

请注意`ml_func`几次引用了`myglob`。在转换为x86汇编时，这将涉及到一个`mov`指令，用于将`myglob`在内存中的值移到一个寄存器中。`mov`指令需要一个绝对地址 - 那么链接器如何知道应该放置哪个地址？答案是 - 它不知道。正如我上面提到的，共享库没有预定义的加载地址 - 这将在运行时决定。

在Linux中，动态加载器是负责准备程序运行的一段代码。它的任务之一是在运行的可执行文件请求时，将共享库从磁盘加载到内存中。当共享库加载到内存中时，它将根据新确定的加载位置进行调整。动态加载器的任务就是解决前一段中提到的问题。

在Linux ELF共享库中，有两种主要方法来解决这个问题：

1. 加载时间重定位（Load-time relocation）

2. 位置无关代码（PIC）

尽管PIC是更常见和现在推荐的解决方案，但在本文中，我将重点介绍加载时间重定位。最终，我计划涵盖这两种方法，并在后面撰写一篇关于PIC的独立文章。

### 加载时重定位和链接动态库

为了创建一个在加载时需要重定位的共享库，我将在编译时使用没有 `-fPIC`标志（否则会触发 PIC 生成）：

```bash
gcc -g -c ml_main.c -o ml_mainreloc.o
gcc -shared -o libmlreloc.so ml_mainreloc.o
```

第一个有趣的事情是看到`libmlreloc.so`的入口点:

```bash
$ readelf -h libmlreloc.so
ELF Header:
  Magic:   7f 45 4c 46 01 01 01 00 00 00 00 00 00 00 00 00
  Class:                             ELF32
  [...] some header fields
  Entry point address:               0x3b0
  [...] some header fields
```

为了简单起见，链接器只是将共享对象链接到地址`0x0`（``.text`节从`0x3b0`开始），知道加载器无论如何都会将其移动。记住这个事实会在本文后面派上用场。

现在让我们来看一下共享库的反汇编，重点关注`ml_func`函数：

```bash
$ objdump -d -Mintel libmlreloc.so

libmlreloc.so:     file format elf32-i386

[...] skipping stuff

0000046c <ml_func>:
 46c: 55                      push   ebp
 46d: 89 e5                   mov    ebp,esp
 46f: a1 00 00 00 00          mov    eax,ds:0x0
 474: 03 45 08                add    eax,DWORD PTR [ebp+0x8]
 477: a3 00 00 00 00          mov    ds:0x0,eax
 47c: a1 00 00 00 00          mov    eax,ds:0x0
 481: 03 45 0c                add    eax,DWORD PTR [ebp+0xc]
 484: 5d                      pop    ebp
 485: c3                      ret

[...] skipping stuff
```

在前两条指令的起始代码`prologue`之后，我们看到了`myglob += a`的编译版本。`myglob`的值从内存中取出到`eax`寄存器中，与`a`（位于`ebp+0x8`处）相加，然后再放回内存中。

但是等等，`mov`指令使用了`myglob`？为什么？

实际上，`mov`指令的操作数只是`0x0`。这是如何工作的？这就是重定位的工作原理。链接器将一些临时的预定义值（在本例中为`0x0`）放入指令流中，然后创建一个指向这个位置的特殊重定位条目。让我们检查一下这个共享库的重定位条目：

```bash
$ readelf -r libmlreloc.so

Relocation section '.rel.dyn' at offset 0x2fc contains 7 entries:
 Offset     Info    Type            Sym.Value  Sym. Name
00002008  00000008 R_386_RELATIVE
00000470  00000401 R_386_32          0000200C   myglob
00000478  00000401 R_386_32          0000200C   myglob
0000047d  00000401 R_386_32          0000200C   myglob
[...] skipping stuff
```

ELF的`rel.dyn`节专用于动态（加载时）重定位，供动态加载器使用。在上面显示的节中，对于`myglob`有3个重定位条目，因为在反汇编中有3个对`myglob`的引用。让我们解读第一个重定位条目:

它说：转到这个对象（共享库）中的偏移量0x470，并对符号`myglob`应用类型为`R_386_32`的重定位。如果我们查阅ELF规范，我们会发现重定位类型`R_386_32`的含义是：将条目中指定偏移量处的值与符号的地址相加，并将其放回偏移量中。在对象的偏移量0x470处有什么？回想一下`ml_func`的反汇编中的这个指令：

```bash
46f:  a1 00 00 00 00          mov    eax,ds:0x0
```

`a1`编码了`mov`指令，因此它的操作数从下一个地址开始，即`0x470`。这就是我们在反汇编中看到的`0x0`。因此，回到重定位条目，我们现在看到它说：将`myglob`的地址添加到该`mov`指令的操作数中。换句话说，它告诉动态加载器 - 一旦执行实际地址分配，将`myglob`的真实地址放入`0x470`，从而用正确的符号值替换`mov`指令的操作数。很巧妙，对吧？

请注意重定位部分中的“Sym. value”列，其中`myglob`的值为`0x200C`。这是`myglob`在共享库的虚拟内存图像中的偏移量（需要注意的是，链接器假设共享库加载在0x0）。这个值也可以通过查看库的符号表来检查，例如使用`nm`命令。

```bash
$ nm libmlreloc.so
[...] skipping stuff
0000200c D myglob
```

这个输出还提供了库中`myglob`的偏移量。`D`表示该符号位于初始化数据段（`.data`）中。

### 加载时间重定位的实际操作

为了观察加载时间重定位的实际操作，我将使用一个简单的驱动可执行文件和我们的共享库。在运行这个可执行文件时，操作系统将加载共享库并适当地进行重定位。

然而，由于Linux启用了地址空间布局随机化（[ASLR](https://en.wikipedia.org/wiki/Address_space_layout_randomization)）功能，重定位相对较难跟踪，因为每次运行可执行文件时，`libmlreloc.so`共享库都会被放置在不同的虚拟内存地址上(每次运行时，调用ldd命令检查可执行文件会报告不同的共享库加载地址)

不过，这只是一个相对较弱的阻碍。我们有办法来解决。但首先，让我们来讨论一下我们的共享库包含的段：

```bash
$ readelf --segments libmlreloc.so

Elf file type is DYN (Shared object file)
Entry point 0x3b0
There are 6 program headers, starting at offset 52

Program Headers:
  Type           Offset   VirtAddr   PhysAddr   FileSiz MemSiz  Flg Align
  LOAD           0x000000 0x00000000 0x00000000 0x004e8 0x004e8 R E 0x1000
  LOAD           0x000f04 0x00001f04 0x00001f04 0x0010c 0x00114 RW  0x1000
  DYNAMIC        0x000f18 0x00001f18 0x00001f18 0x000d0 0x000d0 RW  0x4
  NOTE           0x0000f4 0x000000f4 0x000000f4 0x00024 0x00024 R   0x4
  GNU_STACK      0x000000 0x00000000 0x00000000 0x00000 0x00000 RW  0x4
  GNU_RELRO      0x000f04 0x00001f04 0x00001f04 0x000fc 0x000fc R   0x1

 Section to Segment mapping:
  Segment Sections...
   00     .note.gnu.build-id .hash .gnu.hash .dynsym .dynstr .gnu.version .gnu.version_r .rel.dyn .rel.plt .init .plt .text .fini .eh_frame
   01     .ctors .dtors .jcr .dynamic .got .got.plt .data .bss
   02     .dynamic
   03     .note.gnu.build-id
   04
   05     .ctors .dtors .jcr .dynamic .got
```

为了追踪myglob符号，我们对此处列出的第二个段感兴趣。请注意以下几点：

+ 在底部的“section to segment mapping”中，表示段`01`包含了`.data`节，该节是`myglob`的位置。

+ `VirtAddr`列指定了第二个段的起始地址为`0x1f04`，大小为`0x10c`，意味着它延伸到了`0x2010`，因此包含了位于`0x200C`处的`myglob`。

现在让我们使用Linux提供给我们的一个很好的工具来检查加载时链接过程[`dl_iterate_phdr`](https://linux.die.net/man/3/dl_iterate_phdr)函数。该函数允许应用程序在运行时查询自己已加载的共享库，更重要的是可以查看它们的程序头部。因此，我将以下代码写入`driver.c`文件中：

```cpp
#define _GNU_SOURCE
#include <link.h>
#include <stdlib.h>
#include <stdio.h>


static int header_handler(struct dl_phdr_info* info, size_t size, void* data)
{
    printf("name=%s (%d segments) address=%p\n",
            info->dlpi_name, info->dlpi_phnum, (void*)info->dlpi_addr);
    for (int j = 0; j < info->dlpi_phnum; j++) {
         printf("\t\t header %2d: address=%10p\n", j,
             (void*) (info->dlpi_addr + info->dlpi_phdr[j].p_vaddr));
         printf("\t\t\t type=%u, flags=0x%X\n",
                 info->dlpi_phdr[j].p_type, info->dlpi_phdr[j].p_flags);
    }
    printf("\n");
    return 0;
}


extern int ml_func(int, int);


int main(int argc, const char* argv[])
{
    dl_iterate_phdr(header_handler, NULL);

    int t = ml_func(argc, argc);
    return t;
}
```

`header_handler`是实现了`dl_iterate_phdr`的回调函数。它将被调用以获取所有库的名称和加载地址，以及它们所有的段。

这段代码它还调用了来自libmlreloc.so共享库的ml_func函数。

跑一下编译：

```cpp
gcc -g -c driver.c -o driver.o
gcc -o driver driver.o -L. -lmlreloc
```

独立运行驱动程序时我们可以获取信息，但每次运行时地址都不同。因此，我将在`gdb`下运行它，查看它的输出，然后使用`gdb`进一步查询进程的内存空间。

```bash
 $ gdb -q driver
 Reading symbols from driver...done.
 (gdb) b driver.c:31
 Breakpoint 1 at 0x804869e: file driver.c, line 31.
 (gdb) r
 Starting program: driver
 [...] skipping output
 name=./libmlreloc.so (6 segments) address=0x12e000
                header  0: address=  0x12e000
                        type=1, flags=0x5
                header  1: address=  0x12ff04
                        type=1, flags=0x6
                header  2: address=  0x12ff18
                        type=2, flags=0x6
                header  3: address=  0x12e0f4
                        type=4, flags=0x4
                header  4: address=  0x12e000
                        type=1685382481, flags=0x6
                header  5: address=  0x12ff04
                        type=1685382482, flags=0x4

[...] skipping output
 Breakpoint 1, main (argc=1, argv=0xbffff3d4) at driver.c:31
 31    }
 (gdb)
```

由于驱动程序报告了它加载的所有库（甚至是隐式加载的库，如`libc`或动态加载器本身），输出内容很长，我将只关注关于`libmlreloc.so`的报告。请注意，这6个段与`readelf`报告的段是相同的，但这次它们已经根据其最终的内存位置进行了重定位。

我们来进行一些计算。输出中显示`libmlreloc.so`被放置在虚拟地址`0x12e000`。我们对第二个段感兴趣，正如我们在`readelf`中看到的，它位于偏移量`0x1f04`。确实，我们在输出中看到它被加载到地址`0x12ff04`处。由于`myglob`位于文件中的偏移量`0x200c`处，我们预计它现在的地址将是`0x13000c`。因此，让我们询问`GDB`：

```bash
(gdb) p &myglob
$1 = (int *) 0x13000c
```

太好了！但是关于引用`myglob`的`ml_func`函数的代码呢？我们再次向`GDB`询问一下：

```bash
(gdb) set disassembly-flavor intel
(gdb) disas ml_func
Dump of assembler code for function ml_func:
   0x0012e46c <+0>:   push   ebp
   0x0012e46d <+1>:   mov    ebp,esp
   0x0012e46f <+3>:   mov    eax,ds:0x13000c
   0x0012e474 <+8>:   add    eax,DWORD PTR [ebp+0x8]
   0x0012e477 <+11>:  mov    ds:0x13000c,eax
   0x0012e47c <+16>:  mov    eax,ds:0x13000c
   0x0012e481 <+21>:  add    eax,DWORD PTR [ebp+0xc]
   0x0012e484 <+24>:  pop    ebp
   0x0012e485 <+25>:  ret
End of assembler dump.
```

正如预期的那样，`myglob`的真实地址被放置在所有引用它的`mov`指令中，就像重定位条目所指定的那样。

### 函数调用重定位

到目前为止，本文演示了数据引用的重定位，以全局变量myglob为例。另一个需要重定位的是代码引用，也就是函数调用。本节是关于如何完成这一过程的简要指南。由于现在可以假设读者已经理解了重定位的概念，因此这部分的进程要快得多。

废话不多说，我们开始吧。我修改了共享库的代码如下：

```bash
int myglob = 42;

int ml_util_func(int a)
{
    return a + 1;
}

int ml_func(int a, int b)
{
    int c = b + ml_util_func(a);
    myglob += c;
    return b + myglob;
}
```

添加了`ml_util_func`函数，并且它在`ml_func`中被调用。以下是链接共享库中`ml_func`的反汇编代码：

```bash
000004a7 <ml_func>:
 4a7:   55                      push   ebp
 4a8:   89 e5                   mov    ebp,esp
 4aa:   83 ec 14                sub    esp,0x14
 4ad:   8b 45 08                mov    eax,DWORD PTR [ebp+0x8]
 4b0:   89 04 24                mov    DWORD PTR [esp],eax
 4b3:   e8 fc ff ff ff          call   4b4 <ml_func+0xd>
 4b8:   03 4
 
 5 0c                add    eax,DWORD PTR [ebp+0xc]
 4bb:   89 45 fc                mov    DWORD PTR [ebp-0x4],eax
 4be:   a1 00 00 00 00          mov    eax,ds:0x0
 4c3:   03 45 fc                add    eax,DWORD PTR [ebp-0x4]
 4c6:   a3 00 00 00 00          mov    ds:0x0,eax
 4cb:   a1 00 00 00 00          mov    eax,ds:0x0
 4d0:   03 45 0c                add    eax,DWORD PTR [ebp+0xc]
 4d3:   c9                      leave
 4d4:   c3                      ret
```

这里有趣的是地址`0x4b3`处的指令-它是对`ml_util_func`的调用。

让我们仔细解析一下：`e8`是`call`指令的操作码。该调用的参数是相对于下一条指令的偏移量。在上面的反汇编中，此参数为0xfffffffc，或者简单地说是-4。因此，该调用当前指向了自身。显然，这是不对的-但别忘了重定位。下面是共享库的重定位部分的样子：

```cpp
$ readelf -r libmlreloc.so

Relocation section '.rel.dyn' at offset 0x324 contains 8 entries:
 Offset     Info    Type            Sym.Value  Sym. Name
00002008  00000008 R_386_RELATIVE
000004b4  00000502 R_386_PC32        0000049c   ml_util_func
000004bf  00000401 R_386_32          0000200c   myglob
000004c7  00000401 R_386_32          0000200c   myglob
000004cc  00000401 R_386_32          0000200c   myglob
[...] skipping stuff

```

如果我们将其与先前使用`readelf -r`命令的结果进行比较，我们会注意到增加了针对`ml_util_func`的新条目。该条目指向地址`0x4b4`，这是`call`指令的参数，并且其类型是`R_386_PC32`。这种重定位类型比`R_386_32`要复杂一些，但差别不大。

它的意思如下：获取条目中指定偏移量处的值，加上符号的地址，减去偏移量本身的地址，然后将其放回到偏移量处的字中。请记住，这个重定位是在加载时进行的，那时符号和重定位偏移量的最终加载地址已经知道。这些最终地址参与计算

这样做的目的是什么？基本上，这是一种相对重定位，考虑到其自身的位置，因此适用于具有相对定址的指令的参数（如`e8`调用）。我保证一旦我们看到真正的数字，这一切将变得更清楚。现在，我将构建驱动程序的代码，并再次在GDB中运行它，以查看这种重定位的效果。这是GDB会话，然后是一些解释：

```bash
 $ gdb -q driver
 Reading symbols from driver...done.
 (gdb) b driver.c:31
 Breakpoint 1 at 0x804869e: file driver.c, line 31.
 (gdb) r
 Starting program: driver
 [...] skipping output
 name=./libmlreloc.so (6 segments) address=0x12e000
               header  0: address=  0x12e000
                       type=1, flags=0x5
               header  1: address=  0x12ff04
                       type=1, flags=0x6
               header  2: address=  0x12ff18
                       type=2, flags=0x6
               header  3: address=  0x12e0f4
                       type=4, flags=0x4
               header  4: address=  0x12e000
                       type=1685382481, flags=0x6
               header  5: address=  0x12ff04
                       type=1685382482, flags=0x4

[...] skipping output
Breakpoint 1, main (argc=1, argv=0xbffff3d4) at driver.c:31
31    }
(gdb)  set disassembly-flavor intel
(gdb) disas ml_util_func
Dump of assembler code for function ml_util_func:
   0x0012e49c <+0>:   push   ebp
   0x0012e49d <+1>:   mov    ebp,esp
   0x0012e49f <+3>:   mov    eax,DWORD PTR [ebp+0x8]
   0x0012e4a2 <+6>:   add    eax,0x1
   0x0012e4a5 <+9>:   pop    ebp
   0x0012e4a6 <+10>:  ret
End of assembler dump.
(gdb) disas /r ml_func
Dump of assembler code for function ml_func:
   0x0012e4a7 <+0>:    55     push   ebp
   0x0012e4a8 <+1>:    89 e5  mov    ebp,esp
   0x0012e4aa <+3>:    83 ec 14       sub    esp,0x14
   0x0012e4ad <+6>:    8b 45 08       mov    eax,DWORD PTR [ebp+0x8]
   0x0012e4b0 <+9>:    89 04 24       mov    DWORD PTR [esp],eax
   0x0012e4b3 <+12>:   e8 e4 ff ff ff call   0x12e49c <ml_util_func>
   0x0012e4b8 <+17>:   03 45 0c       add    eax,DWORD PTR [ebp+0xc]
   0x0012e4bb <+20>:   89 45 fc       mov    DWORD PTR [ebp-0x4],eax
   0x0012e4be <+23>:   a1 0c 00 13 00 mov    eax,ds:0x13000c
   0x0012e4c3 <+28>:   03 45 fc       add    eax,DWORD PTR [ebp-0x4]
   0x0012e4c6 <+31>:   a3 0c 00 13 00 mov    ds:0x13000c,eax
   0x0012e4cb <+36>:   a1 0c 00 13 00 mov    eax,ds:0x13000c
   0x0012e4d0 <+41>:   03 45 0c       add    eax,DWORD PTR [ebp+0xc]
   0x0012e4d3 <+44>:   c9     leave
   0x0012e4d4 <+45>:   c3     ret
End of assembler dump.
(gdb)
```

注意这些要点：

1. 在驱动程序的输出中，我们可以看到`libmlreloc.so`的第一个段（代码段）已映射到`0x12e000`(又是这个地址，证明gdb是可以关掉动态加载的)

2. `ml_util_func`被加载到地址`0x0012e49c`

3. 重定位偏移量的地址是0x0012e4b4

4. `ml_func`中对`ml_util_func`的调用被修补为在参数中放置`0xffffffe4`（我使用/r标志来对`ml_func`进行反汇编，以显示原始的十六进制代码以及反汇编指令），这相当于对`ml_util_func`的正确偏移量。

显然，我们最感兴趣的是如何实现4。再次进行一些数学计算。根据前面提到的`R_386_PC32`重定位条目进行解释，我们有：获取条目中指定偏移量处的值（0xfffffffc），加上符号的地址（0x0012e49c），减去偏移量本身的地址（0x0012e4b4），然后将其放回到偏移量处的字中。当然，所有的计算都是基于32位二进制补码进行的。结果如预期，为0xffffffe4。

### 为什么需要调用重定位？

讨论了Linux中共享库加载实现的一些特殊情况。如果你只是想理解如何进行重定位，可以安全地跳过这部分。

在尝试理解对ml_util_func进行的调用重定位时，我必须承认我思考了一段时间。回想一下，call的参数是一个相对偏移量。当库被加载时，call和ml_util_func之间的偏移量肯定不会改变 - 它们都在代码段中，它作为一个整体进行移动。那么为什么需要进行重定位呢？

这里有一个小实验可以尝试：返回到共享库的代码，将ml_util_func声明的声明中添加static关键字。重新编译并再次查看readelf -r的输出。

完成了吗？无论如何，我将揭示结果 - 重定位消失了！查看ml_func的反汇编 - 现在有一个正确的偏移量作为call的参数 - 不需要进行重定位。发生了什么？

在将全局符号引用与其实际定义关联时，动态加载器对共享库的搜索顺序有一些规则。用户还可以通过设置LD_PRELOAD环境变量来影响此顺序。

这里有太多的细节需要涉及，所以如果你真的有兴趣，你需要查看ELF标准、动态加载器的手册，并进行一些搜索。简而言之，当ml_util_func是全局的时，它可能会在可执行文件或其他共享库中被覆盖，因此在链接我们的共享库时，链接器不能假设偏移量是已知的并将其硬编码。它使所有对全局符号的引用都是可重定位的，以便让动态加载器决定如何解析它们。这就是为什么声明函数为静态的会有所不同-因为它不再是全局或导出的，链接器可以在代码中硬编码它的偏移量。

### 从可执行文件引用共享库中的数据

在上面的示例中，myglob仅在共享库内部使用。如果我们从程序（driver.c）中引用它会发生什么呢？毕竟，myglob是一个全局变量，因此在外部是可见的。

让我们将driver.c修改如下（注意，我已经删除了段迭代的代码）：

```cpp
#include <stdio.h>

extern int ml_func(int, int);
extern int myglob;

int main(int argc, const char* argv[])
{
    printf("addr myglob = %p\n", (void*)&myglob);
    int t = ml_func(argc, argc);
    return t;
}
```

mayglob的地址：

```bash
addr myglob = 0x804a018
```

等等，这里有些不对劲。myglob不是在共享库的地址空间中吗？0x804xxxx看起来像是程序的地址空间。出了什么问题？

请记住，程序/可执行文件是不可重定位的，因此其数据地址必须在链接时进行绑定。因此，链接器必须在程序的地址空间中创建变量的副本，并且动态加载器将使用该副本作为重定位地址。这类似于先前部分中的讨论-在某种程度上，主程序中的myglob覆盖了共享库中的变量，并且根据全局符号查找规则，它被作为使用的变量。如果我们在GDB中检查ml_func，我们将看到对myglob的正确引用：

```bash
0x0012e48e <+23>:      a1 18 a0 04 08 mov    eax,ds:0x804a018
```

这是有道理的，因为libmlreloc.so中仍然存在对myglob的R_386_32重定位，而动态加载器会将它指向myglob现在所在的正确位置

这一切都很好，但还有一个问题。myglob是在共享库中初始化的（为42）-如何将这个初始化值传递到程序的地址空间中？实际上，链接器会在程序中构建一个特殊的重定位条目（到目前为止，我们只关注了共享库中的重定位条目）：

```bash
$ readelf -r driver

Relocation section '.rel.dyn' at offset 0x3c0 contains 2 entries:
 Offset     Info    Type            Sym.Value  Sym. Name
08049ff0  00000206 R_386_GLOB_DAT    00000000   __gmon_start__
0804a018  00000605 R_386_COPY        0804a018   myglob
[...] skipping stuff
```

请注意myglob的R_386_COPY重定位。它的含义很简单：将符号的值从其地址复制到该偏移量中。动态加载器在加载共享库时执行此操作。它如何知道要复制多少？符号表部分包含每个符号的大小；例如，在libmlreloc.so的.symtab部分中，myglob的大小为4。我认为这是一个非常酷的例子，展示了执行链接和加载的过程是如何协同进行的。链接器在输出中放置了特殊的指令，供动态加载器消费和执行。

我认为这是一个非常棒的例子，展示了可执行文件的链接和加载过程是如何协同进行的。链接器在输出中放置了特殊的指令，供动态加载器消费和执行。

## 总结

加载时重定位是在Linux（以及其他操作系统）中在将共享库加载到内存时解决内部数据和代码引用的方法之一。如今，位置无关代码（PIC）是一种更受欢迎的方法，一些现代系统（如x86-64）已经不再支持加载时重定位。

然而，我决定写一篇关于加载时重定位的文章有两个原因。首先，在某些系统上，加载时重定位相比PIC具有一些优势，特别是在性能方面。其次，我认为在没有先验知识的情况下，加载时重定位更容易理解，这将使得以后更容易解释PIC。

无论出于什么动机，我希望本文可以帮助揭示在现代操作系统中链接和加载共享库的幕后魔术。

## Ref

1. [Load-time relocation of shared libraries](https://eli.thegreenplace.net/2011/08/25/load-time-relocation-of-shared-libraries)

2. [x86 mem layout](https://eli.thegreenplace.net/2011/02/04/where-the-top-of-the-stack-is-on-x86/)

3. [x-hook](https://github.com/iqiyi/xHook/blob/master/docs/overview/android_plt_hook_overview.zh-CN.md#phtprogram-header-table)

4. [process addresses and entry point](https://eli.thegreenplace.net/2011/01/27/how-debuggers-work-part-2-breakpoints)

5. [Shared_library](https://en.wikipedia.org/wiki/Shared_library)

6. [Position Independent Code (PIC) in shared libraries](https://eli.thegreenplace.net/2011/11/03/position-independent-code-pic-in-shared-libraries/)

7. [C/C++动态链接及地址无关代码(PIC)](https://zhuanlan.zhihu.com/p/589437215)

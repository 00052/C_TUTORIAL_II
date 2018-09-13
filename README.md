# C Tutorial II

## 参考文档
[GCC online documentation](https://gcc.gnu.org/onlinedocs/)
[MSVS 2017 Organization of the C Language Reference](https://docs.microsoft.com/en-us/cpp/c-language/organization-of-the-c-language-reference?view=vs-2017)

## 说明
最近看到Github上各种语言的基础教程和引导，对于初学者来说，真的是入门过坑少走弯路的理想学习资源。所以我也插个队，来凑凑热闹，在这里做一些总结，希望这些引导能让刚入门的同学有一个快速的提高过程。

如你所愿，《C Tutorial I》是不存在的，还没有入门的同学可能不太适合看这个教程，建议先补习以下基础。

这篇教程从编译器讲起，我会尽可能使用精简的语言进行讲解，分享更多的干货，更偏重实用性和整体知识脉络的把握，对于细节可参考上述实现文档。

由于本人能力有限，对C的掌握还不够精深。如果在阅读中发现错误或疏漏，欢迎提交ISSUE。

## 编译器(Compiler)
通常我们使用的C编译器有三种

* *Visual Studio(MSVC)系列*  
  由Microsoft开发的基于Win32的闭源编译器实现
* *GCC*  
   Richard Stallman 发起GNU计划中的基础编译套件，现已广泛移植到各大主流操作系统和平台，是Linux各发行版的默认编译器，使用GPL协议发布。
* *Clang*  
   由Apple开发的面向OSX的编译器实现，现已移植到各大主流操作系统,使用BSD协议发布。

后续教程中，我们会使用到 GCC和MSVC。

## 目标构建

在通用操作系统中，从C源代码构建目标可执行文件(或称执行档)主要包含4个流程：预处理、编译、汇编和链接。

在Linux系统中使用Shell环境创建源文件donothinig.c，当然你可以选择Bash或zsh，笔者将使用Bash作为演示环境。  


```C
int main() {}
```

编译源文件 donothing.c

```sh
$ gcc donothing.c
```
生成名为a.out 的可执行文件，使用file命令查看文件格式

```sh
$ file a.out 
a.out: ELF 64-bit LSB executable,  ...  not stripped
```
可知a.out格式为64位的 ELF(Executable and Linkable Format)可执行格式，这是类UNIX中通用的执行档格式。LSB表示小端序，指该可执行文件的目标处理器架构是小端序（如ix86、arm处理器），而对于MIPS等大端序处理器，用MSB表示。命令 `gcc donothing.c` 自动执行了上述构建过程的全部，在不指定输出文件名的情况下，默认是 **a.out **。

下面分析构建目标的四个过程

### 预处理 (preprocess)
GNU 提供了预处理命令`cpp`，它的全称是The C Preprocessor，我们使用 helloworld 程序 `hello.c' 做示例，来分析预处理的行为。

```C
/* hello.c */
#include <stdio.h>
int main(){
    printf("Hello world\n");
	return 0;
}
```
进行预处理
```sh
$ cpp -P hello.c hello.i
```
执行完毕，在当前目录生成 hello.i，打开 hello.i，你会惊奇的发现，`#include <stdio.h>` 消失了，取而代之的是`extern`，`typedef`和其他形式的声明。
这其中包含了最重要的一行语句，就是`printf`函数的声明：
```C
extern int printf (const char *__restrict __format, ...);
```
这中间到底发生了什么？其实预处理就是一个预处理语句的展开过程，常见的预处理语句包括`#define`、`#if...#endif`、`include`、`#line`等，预处理器会递归展开这些语句，最终替换为干净的C程序，不包含任何符号`#‘ 开头的行。（注：cpp命令的 `-P’ 参数会移除预处理文件中的linemarker行，这些行是以符号`#'开头的，这不在文章讨论的范围内）

那么我们猜想，如果把文件hello.c的预处理语句`#include <stdio.h>` 替换为上述printf函数声明。是不是可以被正确编译执行呢？
```C
/* hello.c */
extern int printf (const char *__restrict __format, ...);
int main(){
    printf("Hello world\n");
	return 0;
}
```
答案是肯定的
```sh
$ gcc hello.c
$ ./a.out
Hello world
```
printf函数在其声明的作用下成功定向到了C库printf函数实现的入口地址。

### 编译(rompile)
将上一步产生的预处理文件 hello.i 进行编译
```sh
$ gcc -S hello.i
```
此时在当前目录会生成同名但是后缀为.s的文件（你可以通过  `-o <file>' 参数来指定其他的输出文件名）。这个文件是AT&T格式的GAS汇编文件，文件内容如下：（注：笔者当前使用的OS是64位LinuxMint19发行版)
```asm
	.file	"hello.i"
	.globl	s
	.section	.rodata
.LC0:
	.string	"helloworld\n"
	.data
	.align 8
	.type	s, @object
	.size	s, 8
s:
	.quad	.LC0
	.text
	.globl	main
	.type	main, @function
main:
.LFB0:
	.cfi_startproc
	pushq	%rbp
	.cfi_def_cfa_offset 16
	.cfi_offset 6, -16
	movq	%rsp, %rbp
	.cfi_def_cfa_register 6
	movq	s(%rip), %rax
	movq	%rax, %rdi
	movl	$0, %eax
	call	printf
	movl	$63, %eax
	popq	%rbp
	.cfi_def_cfa 7, 8
	ret
	.cfi_endproc
.LFE0:
	.size	main, .-main
	.ident	"GCC: (Ubuntu 5.4.0-6ubuntu1~16.04.10) 5.4.0 20160609"
	.section	.note.GNU-stack,"",@progbits
```
如果你同笔者一样，觉得AT&T汇编真的很难看，你可以通过为`gcc`命令添加`-masm=intel'选项来产生Intel格式的汇编代码。
```asm
	.file	"hello.i"
	.intel_syntax noprefix
	.section	.rodata
.LC0:
	.string	"Hello world"
	.text
	.globl	main
	.type	main, @function
main:
.LFB0:
	.cfi_startproc
	push	rbp
	.cfi_def_cfa_offset 16
	.cfi_offset 6, -16
	mov	rbp, rsp
	.cfi_def_cfa_register 6
	mov	edi, OFFSET FLAT:.LC0
	call	puts
	mov	eax, 0
	pop	rbp
	.cfi_def_cfa 7, 8
	ret
	.cfi_endproc
.LFE0:
	.size	main, .-main
	.ident	"GCC: (Ubuntu 5.4.0-6ubuntu1~16.04.10) 5.4.0 20160609"
	.section	.note.GNU-stack,"",@progbits
```
*掌握i386处理器及其64位扩展汇编应该是是一个C程序员基本的技能要求*

很显然，printf函数调用，在编译优化过程中被替换为执行效率更高的puts函数，汇编程序在此不做过多解释，存在阅读困难的同学，可以自行学习一下相关内容。其中一行关键指令`.globl  main`，随后我们会用到，.global 是一个汇编器伪指令，用来标明将main函数首地址以符号形式导出（或理解为暴露），以便被其他模块调用。
### 汇编 (assemble)
GNU编译套件提供了`as`命令，对GAS格式文件进行汇编生成可重定位目标文件，这种文件二进制文件并且不能被直接执行，它包含了机器码、调试信息、数据和重定位信息。下面使用`as`命令对上一步生成的文件hello.s进行汇编操作
```sh
$ as -o hello.o hello.s
```
该命令会产生hello.o文件，使用`file`命令查看文件格式
```sh
$ file hello.o
hello.o: ELF 64-bit LSB relocatable, x86-64, version 1 (SYSV), not stripped
```
这表明生成的文件格式为64位的ELF小端序可重定位目标文件，该种格式文件可以作为模块与其他模块被链接器捆绑在一起，构成单独的可执行程序。

使用GNU开发者工具中的`nm`命令查看该文件的符号表

```sh
$ nm hello.o
0000000000000000 T main
                 U puts
```
`T'表示代码段（text section)中导出的符号，前面的64位数值表示基于段首的地址偏移，它是用16进制表示的。main函数在最终的可执行程序中，实际被CRT（C运行时环境）调用。

`U'未实现的符号，因为puts符号需要从C库中动态导入，在后续的链接中将被重定向。

### 链接(link)
链接操作是通过链接器（linker），将一个或多个（可重定位）目标文件合并为一个可执行文件。  
类UNIX上常用的链接器是GNU ld，而MSVC链接器是 link.exe。

尝试使用gcc命令将hello.o链接为可执行文件并执行
```sh
$ gcc -o hello hello.o
$ ./hello
Hello world
```
字符串 'Hello world' 被正确输出，显然hello.o被成功链接为可执行文件。但这可能并不是我们想看到的链接方式，所有复杂的链接细节都被隐藏了。
下面使用GNU ld改写上述链接过程
```sh
$ ld -dynamic-linker /lib64/ld-linux-x86-64.so.2\
>    -o hello                                   \
>    /usr/lib/x86_64-linux-gnu/crt1.o           \
>    /usr/lib/x86_64-linux-gnu/crti.o           \
>    hello.o                                    \
>    /usr/lib/x86_64-linux-gnu/crtn.o           \
>    -lc
```
对于这条看似古怪的命令行，可能有些读者会产生不适，不过不用担心，它实际并不复杂。

-dynamic-linker <file> 参数指定动态链接器，它是一个共享库(Shared library)，在ELF可执行文件加载并映射到内存后，内核将控制权转交给动态链接器。动态链接器会分析可执行文件动态段(Dynamic section)，将依赖的共享库映射到进程虚拟地址空间，再针对共享库重复上一步工作，直到将需要的共享库全部加载，然后对可执行文件和需要的共享库执行重定向，提供对共享库初始化和卸载清理的执行环境，然后将控制权转交给程序。
'-lc' 参数会动态链接C库 /lib/x86\_64-linux-gnu/libc.so.6，它并不会把libc.so.6文件链接到执行档，而是把运行时依赖关系写入动态段。使用objdump命令可查看ELF执行档依赖的共享库  
```sh
objdump -p hello | grep NEEDED
  NEEDED               libc.so.6
```

crt1.o crti.o crtn.o 是建立C运行时（CRT）环境基本的模块，CRT主要作用是提供进入main函数前的初始化和main函数返回后的清理工作。crt1.o包含程序入口点\_start和两个关键导入符号\_\_libc\_start\_main和main，\_\_libc\_start\_main负责初始化LIBC及调用main函数，crti.o和ctrn.o共同包含了两个段.init、.fini，这两个段的入口是符号\_init和\_fini会在main函数执行前与返回后被先后调用。用来初始化C++的全局静态对象。

### 在MSVC下的编译流程
...

## 堆栈
...


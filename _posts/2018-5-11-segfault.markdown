---
layout:     post
title:      "如何锁定 Segmentation Fault 的病灶"
subtitle:   "告诉我,哪条语句segmentation fault了？"
date:       2018-5-11 12:00:00
author:     "Refone"
header-img: "img/glass.jpg"
tags:
    - C
    - Debug
---

当在一个巨大的工程运行时，突然来了一个Segmentation Fault, 形如:
```bash
[1]    1776 segmentation fault   ./hello
```
怎么办？
下意识查一下dmesg，会看到真有Error信息：
```bash
[768977.271941] hello[1480]: segfault at 5652202637c5 ip 000056522026370b sp 00007ffe46ac7ae0 error 7 in hello[565220263000+1000]
```
不过这样的信息并不能锁定到底是哪一行代码出了问题。

那么现在我们有如下方法：

* 用GDB对hello进行单步调试，锁定病灶。
* 在hello.c中插入print，看看到底会挂在哪里，灵性一点还可以利用二分查找
* 利用这行dmesg信息

前两种我们就不谈了，小程序这样调一调bug就算了，大项目运行一次说不定就是几十分钟甚至若干小时，单步调试可以调一年，更不用说每次加打印运行了，人只能活几十年。

所以，**利用好宝贵的每一条error信息非常重要**。

接下来我们来模拟这样一个错误：

#### 1.创建存在segfault的c程序
hello.c :
```c
#include <stdio.h>
#include <stdlib.h>

int main()
{
    char *c = "hello world.";
    c[1] = 'H';

    printf("done.\n");
    return 0;
}
```
c程序就简单如上了，我们可以知道```c[1] = 'H'```会发生segment fault，因为```"hello world"```是一个常量字符串，```c```指向的是一个常量字符串，不能对它进行write access。

#### 2.build
接下来我们用以下的命令进行build:
```bash
gcc -g -o hello hello.c
```
值得注意的是我们多了一个```-g```命令，这是用来在编译的时候产生调试信息的选项。

#### 3.运行
果不其然，运行会报segfault
```bash
refone@broadwell: ~/hello
[0] % ./hello
[1]    2101 segmentation fault (core dumped)  ./hello
```
查看dmesg,```sudo dmesg```可以看到如下信息:
```bash
[771630.319476] hello[2101]: segfault at 5626e3ed4775 ip 00005626e3ed46cb sp 00007ffff608c710 error 7 in hello[5626e3ed4000+1000]
```
其中重要的是你知道：

1. 是**hello**进程出了问题。
2. 在**ip 00005626e3ed46cb**的位置上出了问题。

#### 4.锁定病灶
将hello进程的可执行文件反汇编，利用```objdump```指令，添加```-S```(大写)选项以显示源代码。
```bash
objdump -S hello > hello.s
```
在```hello.s```汇编代码中查找ip的最后几位

**注意：** 这个ip每次运行hello都是不同的，但也只是前面的不同，这跟kernel分配进程内存有关，但最后的进程内偏移量却是稳定的，
所以可以从ip最后三位开始查找，重复太多则多加一位。此外，地址随机化需要关闭，这个GCC默认是关闭的，不然你会发现进程内偏移量
也每次都会变。
```x86asm
int main()
{
 6b0:   55                      push   %rbp
 6b1:   48 89 e5                mov    %rsp,%rbp
 6b4:   48 83 ec 10             sub    $0x10,%rsp
    char *c = "hello world.";
 6b8:   48 8d 05 b5 00 00 00    lea    0xb5(%rip),%rax        # 774 <_IO_stdin_used+0x4>
 6bf:   48 89 45 f8             mov    %rax,-0x8(%rbp)
    c[1] = 'H';
 6c3:   48 8b 45 f8             mov    -0x8(%rbp),%rax
 6c7:   48 83 c0 01             add    $0x1,%rax
 6cb:   c6 00 48                movb   $0x48,(%rax)

    printf("done.\n");
 6ce:   48 8d 3d ac 00 00 00    lea    0xac(%rip),%rdi        # 781 <_IO_stdin_used+0x11>
 6d5:   e8 86 fe ff ff          callq  560 <puts@plt>
    return 0;
 6da:   b8 00 00 00 00          mov    $0x0,%eax
}
```
知道在```6cb```的位置出错也就不难定位到是```c[1] = 'H'```这条语句出错了。

### 总结

我们的方式需要干这些事：

1. gcc编译的时候加上-g选项。
2. 获得dmesg信息锁定病灶的ip，主要是进程内偏移量。
3. 用objdump反汇编进程执行文件，加上-S选项。

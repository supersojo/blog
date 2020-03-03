---
title: x86_64 calling conventions
date: 2019-06-10 21:10:10
tags:
- x86_64
- 调用约定
categories:
- 调用约定
---

在分析异常调用栈需要了解参数传递细节，在这里整理一下x86_64调用约定。

## 通过函数的汇编语句分析调用约定

### 栈帧布局
```
.text
 foo:
     push %rbp
     mov %rsp,%rbp
     …
 [retaddr][old rbp]
                  ^
                  |
                rbp/rsp
 8(%rbp): return address
 0(%rbp): old %rbp
```

上面是foo函数的汇编语句，在函数入口处，%rsp指向返回地址，重新设置%rbp指向当前栈帧。

### 局部变量
```
foo:
     push %rbp
     mov %rsp,%rbp
     sub $16,%rsp # local var spaces
     …
 [retaddr][old rbp][local vars]
high addr         ^           ^ low addr
                  |           |
                 rbp         rsp
 -8(%rbp):local vars
```

如果在栈上分配局部变量，通过%rbp访问。-8(%rbp)访问第一个局部变量，依次类推访问其它局部变量。

### 返回地址
```
foo:
     push %rbp
     mov %rsp,%rbp
     sub $16,%rsp # local var spaces
     …
     add $16,%rsp # balance stack
     pop %rbp  # restore stack frame pointer
     ret # return to caller
 [retaddr][old rbp][local vars]
                  ^           ^
                  |           |
                 rbp         rsp
 -8(%rbp):local vars
``` 

在使用完局部变量需要把对应栈空间释放，平衡堆栈，最后ret返回caller处。

### 参数传递
```
call conventions:
 void foo(int a,int b,int c,int d,int e,int f,int g);
   40053c:       6a 07                   pushq  $0x7      # seventh use stack
   40053e:       41 b9 06 00 00 00       mov    $0x6,%r9d # sixth
   400544:       41 b8 05 00 00 00       mov    $0x5,%r8d # fifth
   40054a:       b9 04 00 00 00          mov    $0x4,%ecx # fouth
   40054f:       ba 03 00 00 00          mov    $0x3,%edx # third
   400554:       be 02 00 00 00          mov    $0x2,%esi # second para
   400559:       bf 01 00 00 00          mov    $0x1,%edi # first para
 %rdi ->a
 %rsi ->b
 %rdx ->c
 %rcx ->d
 %r8  ->e
 %r9  ->f
 [parameters][retaddr][old rbp]
                              ^
                              |
                             rbp/rsp    
 8(%rbp): return address
 16(%rbp): the seventh parameter
 24(%rbp): the eighth parameter
```

前6个参数使用寄存器传递，如果还有额外参数则通过堆栈传递。

## 参考资料

1. [X86_calling_conventions](https://en.wikipedia.org/wiki/X86_calling_conventions)
2. [The 64 bit x86 C Calling Convention](https://aaronbloomfield.github.io/pdr/book/x86-64bit-ccc-chapter.pdf)


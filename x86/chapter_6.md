第六章 结构体
===

### 6.1 结构体基础
结构体在运行时的本质是一片按照结构体定义分配的内存。比如如下结构体：
```
struct point {
    int x;
	int y;
};
```
其本质就是分配两个int大小的连续内存，内存的前一个int大小的内容是x，后一个int大小的内容是y。  
来一段简单的例子：
```
#include <stdio.h>

struct point {
    int x;
    int y;
};

main()
{
    struct point p;
    p.x = 1;
    p.y = 2;
}
```
其汇编实现如下：
```
_main:                                  ## @main
        .cfi_startproc
## BB#0:
        pushq   %rbp
Ltmp0:
        .cfi_def_cfa_offset 16
Ltmp1:
        .cfi_offset %rbp, -16
        movq    %rsp, %rbp
Ltmp2:
        .cfi_def_cfa_register %rbp
        xorl    %eax, %eax
        movl    $1, -8(%rbp)
        movl    $2, -4(%rbp)
        popq    %rbp
        retq
        .cfi_endproc
```
其实现是把数字1放入栈顶下偏移量8的位置，把数字2放入栈顶下偏移量4的位置。

那更复杂一点的结构体中的结构体呢？
```
struct point {
    int x;
	int y;
};

struct rect {
    struct point pt1;
	struct point pt2;
}
```
其本质就是分配连续2个struct point的空间也就是4个int。
```
#include <stdio.h>

struct point {
    int x;
    int y;
};

struct rect {
    struct point pt1;
	struct point pt2;
};

main()
{
    struct rect r;
    r.pt1.x = 1;
    r.pt1.y = 2;
    r.pt2.x = 3;
    r.pt2.y = 4;
}
```
其汇编实现如下：
```
_main:                                  ## @main
	.cfi_startproc
## BB#0:
	pushq	%rbp
Ltmp0:
	.cfi_def_cfa_offset 16
Ltmp1:
	.cfi_offset %rbp, -16
	movq	%rsp, %rbp
Ltmp2:
	.cfi_def_cfa_register %rbp
	movl	$1, -32(%rbp)
	movl	$2, -28(%rbp)
	movl	$3, -24(%rbp)
	movl	$4, -20(%rbp)
	movq	-16(%rbp), %rax
	movq	-8(%rbp), %rdx
	popq	%rbp
	retq
	.cfi_endproc
```
其实现就是把4个整数放在了栈顶往下-20后的16byte中，非常简单。其中-8/-16这两个算是完全没用到。

### 6.2 结构体与函数
先看看结构体作为函数返回值的例子：
```
#include <stdio.h>

struct point {
    int x;
    int y;
};

struct point makepoint(int x, int y)
{
    struct point p;
    p.x = x;
    p.y = y;
	return p;
}
```
其汇编实现如下：
```
_makepoint:                             ## @makepoint
	.cfi_startproc
## BB#0:
	pushq	%rbp
Ltmp0:
	.cfi_def_cfa_offset 16
Ltmp1:
	.cfi_offset %rbp, -16
	movq	%rsp, %rbp
Ltmp2:
	.cfi_def_cfa_register %rbp
	movl	%edi, -12(%rbp)
	movl	%esi, -16(%rbp)
	movl	-12(%rbp), %esi
	movl	%esi, -24(%rbp)
	movl	-16(%rbp), %esi
	movl	%esi, -20(%rbp)
	movq	-24(%rbp), %rax
	movq	%rax, -8(%rbp)
	movq	-8(%rbp), %rax
	popq	%rbp
	retq
	.cfi_endproc
```
参数部分先临时放入了-12(%rbp)和-16(%rbp)，然后再把参数x/y拿出来，放入分配给结构体的-20(%rbp)和-24(%rbp)。最关键的返回值来了，就是-24(%rbp)处的8字节数据（q）放入 %rax中，然后rax就是返回值了。也就是因为返回的结构体刚好是两个32位的整数刚好能在64位的rax寄存器中放好，就直接拼拼放入rax返回了。  
那如果是struct rect呢？rax就放不下了：
```
#include <stdio.h>

struct point {
    int x;
    int y;
};

struct rect {
    struct point pt1;
	struct point pt2;
};

struct rect makerect(int x1, int y1, int x2, int y2)
{
    struct rect r;
    r.pt1.x = x1;
    r.pt1.y = y1;
    r.pt2.x = x2;
    r.pt2.y = y2;
    return r;
}
```
其汇编实现如下：
```
_makerect:                              ## @makerect
	.cfi_startproc
## BB#0:
	pushq	%rbp
Ltmp0:
	.cfi_def_cfa_offset 16
Ltmp1:
	.cfi_offset %rbp, -16
	movq	%rsp, %rbp
Ltmp2:
	.cfi_def_cfa_register %rbp
	movl	%edi, -20(%rbp)
	movl	%esi, -24(%rbp)
	movl	%edx, -28(%rbp)
	movl	%ecx, -32(%rbp)
	movl	-20(%rbp), %ecx
	movl	%ecx, -48(%rbp)
	movl	-24(%rbp), %ecx
	movl	%ecx, -44(%rbp)
	movl	-28(%rbp), %ecx
	movl	%ecx, -40(%rbp)
	movl	-32(%rbp), %ecx
	movl	%ecx, -36(%rbp)
	movq	-48(%rbp), %rax
	movq	-40(%rbp), %r8
	movq	%r8, -8(%rbp)
	movq	%rax, -16(%rbp)
	movq	-16(%rbp), %rax
	movq	-8(%rbp), %rdx
	popq	%rbp
	retq
	.cfi_endproc
```
前面参数和放值到结构体的部分一样，主要看看返回值吧。把栈里的连续两个64位的内容放入r8和rax中，然后又从r8和rax挪到了rax和rdx中！一个rect的2个point的4个int被分别合并放入了两个64位寄存器中作为返回值。那如果数据是浮点数或者数据量太大寄存器不够用怎么办？放栈上！具体的实现请参见各平台的方法调用标准！

说完返回值再来说说参数，参数也很类似：
```
#include <stdio.h>

struct point {
    int x;
    int y;
};

struct point addpoint(struct point p1, struct point p2)
{
    p1.x += p2.x;
	p1.y += p2.y;
	return p1;
}
```
其汇编实现如下：
```
_addpoint:                              ## @addpoint
        .cfi_startproc
## BB#0:
        pushq   %rbp
Ltmp0:
        .cfi_def_cfa_offset 16
Ltmp1:
        .cfi_offset %rbp, -16
        movq    %rsp, %rbp
Ltmp2:
        .cfi_def_cfa_register %rbp
        movq    %rdi, -16(%rbp)
        movq    %rsi, -24(%rbp)
        movl    -24(%rbp), %eax
        addl    -16(%rbp), %eax
        movl    %eax, -16(%rbp)
        movl    -20(%rbp), %eax
        addl    -12(%rbp), %eax
        movl    %eax, -12(%rbp)
        movq    -16(%rbp), %rsi
        movq    %rsi, -8(%rbp)
        movq    -8(%rbp), %rax
        popq    %rbp
        retq
        .cfi_endproc
```
两个point参数被放入了rdi和rsi两个64位寄存器中，然后取出加了一把，然后返回。

### 6.3 联合体
联合体是个很神奇的东西，同一片内存里面根据取的方式取出来的内容就不一样，十分具有汇编风格，数据没有类型，决定数据类型的是数据被怎么用！来个例子：
```
#include <stdio.h>

union u_tag {
    int ival;
    float fval;
    char *sval;
};

main() {
    union u_tag u;
    u.ival = 1;
    u.fval = 1.0;
    u.sval = "hello, world";
    int a = u.ival;
    float b = u.fval;
    char *c = u.sval;
}
```
其汇编实现如下：
```
_main:                                  ## @main
        .cfi_startproc
## BB#0:
        pushq   %rbp
Ltmp0:
        .cfi_def_cfa_offset 16
Ltmp1:
        .cfi_offset %rbp, -16
        movq    %rsp, %rbp
Ltmp2:
        .cfi_def_cfa_register %rbp
        xorl    %eax, %eax
        leaq    L_.str(%rip), %rcx
        movss   LCPI0_0(%rip), %xmm0    ## xmm0 = mem[0],zero,zero,zero
        movl    $1, -8(%rbp)
        movss   %xmm0, -8(%rbp)
        movq    %rcx, -8(%rbp)
        movl    -8(%rbp), %edx
        movl    %edx, -12(%rbp)
        movss   -8(%rbp), %xmm0         ## xmm0 = mem[0],zero,zero,zero
        movss   %xmm0, -16(%rbp)
        movq    -8(%rbp), %rcx
        movq    %rcx, -24(%rbp)
        popq    %rbp
        retq
        .cfi_endproc

        .section        __TEXT,__cstring,cstring_literals
L_.str:                                 ## @.str
        .asciz  "hello, world"

        .comm   _u,8,3                  ## @u
```
首先把1放入-8(%rbp)，再把浮点数放入-8(%rbp)，再把指针放入-8(%rbp)，几种不同类型的数据放入了同一个内存空间，当然，后面放的会把前面的覆盖掉！去也相同，从-8(%rbp)这同一个空间里面尝试取出整数浮点数和字符指针。  
还有一个细节，union的大小是根据最大的那个成员的数据类型来的，在放其他较小的成员时空间会略有浪费。

### 6.4 bit-field
bit-field是一个巨牛的东西，它可以极度优化内存使用，不过时间换空间嘛，存取速度也会稍微慢些。来个例子：
```
#include <stdio.h>

main() {
    struct {
	    unsigned int is_keyword:1;
		unsigned int is_extern:1;
		unsigned int is_static:1;
	} flags;
	flags.is_keyword = 1;
	flags.is_extern = 0;
	flags.is_static = 1;
}
```
其汇编实现如下：
```
_main:                                  ## @main
        .cfi_startproc
## BB#0:
        pushq   %rbp
Ltmp0:
        .cfi_def_cfa_offset 16
Ltmp1:
        .cfi_offset %rbp, -16
        movq    %rsp, %rbp
Ltmp2:
        .cfi_def_cfa_register %rbp
        xorl    %eax, %eax
        movb    -8(%rbp), %cl
        andb    $-2, %cl
        orb     $1, %cl
        movb    %cl, -8(%rbp)
        movb    -8(%rbp), %cl
        andb    $-3, %cl
        movb    %cl, -8(%rbp)
        movb    -8(%rbp), %cl
        andb    $-5, %cl
        orb     $4, %cl
        movb    %cl, -8(%rbp)
        popq    %rbp
        retq
        .cfi_endproc
```
这里仅仅只用了一个cl寄存器（8位）就放下了三个标记，而实际上也只用到了3位，巨省空间，但是存数据的时候却用了andb和orb两条指令来算值和一条movb指令来存值，如果用int类型的flag就只会需要一条movl指令即可。
第二章 类型, 操作符, 表达式
===

### 2.1 变量名称
变量本质是一块用来存放数据的存储空间, 而变量名称则是对这块空间所处位置的用于助记的别名. 在汇编中没有变量名称的概念, 只有用来存储数据的寄存器和内存.

### 2.2 数据类型
C语言中只有少量的数据类型, 它们的特征如下:
```
char 字符, 大小1 byte, 实际上是一个8位的整数, 占1字节空间
int  整数, 大小根据平台不同而不同, 一般为32位, 占4字节空间
float 单精度浮点数, 大小根据平台不同而不同, 一般为32位, 占4字节空间
double 双精度浮点数, 大小根据平台不同而不同, 一般为64位, 占8字节空间
```
从char开始看实现吧, C代码如下:
```
    char a = 'a';
```
其汇编代码为(其他代码基本没变, 这里只贴出关键代码):
```
    movb    $97, -1(%rbp)
```
把栈顶下1 byte的内容设为97(字符'a'的数值).

接下来是int, C代码如下:
```
    int a = 1;
```
其汇编代码为:
```
    movl	$1, -4(%rbp)
```
把栈顶下4 byte的内容设为1.

接下来是float, C代码如下:
```
    float a = 1.1;
```
其汇编代码为:
```
        .section        __TEXT,__text,regular,pure_instructions
        .macosx_version_min 10, 12
        .section        __TEXT,__literal4,4byte_literals
        .align  2
LCPI0_0:
        .long   1066192077              ## float 1.10000002
        .section        __TEXT,__text,regular,pure_instructions
        .globl  _main
        .align  4, 0x90
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
        movss   LCPI0_0(%rip), %xmm0    ## xmm0 = mem[0],zero,zero,zero
        movss   %xmm0, -4(%rbp)
        popq    %rbp
        retq
        .cfi_endproc


.subsections_via_symbols
```
可以看出浮点数的变动比较大, 它的数值存在`__TEXT,__literal4`里, 数据对齐为2的2次方(.align 2里面的2代表的是2次方), 数据指令是`.long`(4字节):
```
        .section        __TEXT,__literal4,4byte_literals
        .align  2
LCPI0_0:
        .long   1066192077              ## float 1.10000002
```
而在代码中通过引用把内容加载到浮点数寄存器中, 然后从浮点寄存器再保存到栈中:
```
        movss   LCPI0_0(%rip), %xmm0    ## xmm0 = mem[0],zero,zero,zero
        movss   %xmm0, -4(%rbp)
```
从`-4(%rbp)`可以看出float占4 byte.

接下来是double, C代码如下:
```
    double a = 1.1;
```
其汇编代码如下:
```
        .section        __TEXT,__text,regular,pure_instructions
        .macosx_version_min 10, 12
        .section        __TEXT,__literal8,8byte_literals
        .align  3
LCPI0_0:
        .quad   4607632778762754458     ## double 1.1000000000000001
        .section        __TEXT,__text,regular,pure_instructions
        .globl  _main
        .align  4, 0x90
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
        movsd   LCPI0_0(%rip), %xmm0    ## xmm0 = mem[0],zero
        movsd   %xmm0, -8(%rbp)
        popq    %rbp
        retq
        .cfi_endproc


.subsections_via_symbols
```
主要变动的地方在于, double数值存在`__TEXT,__literal8`中, 数据对齐为2的3次方, 数据指令是`.quad`(8字节):
```
        .section        __TEXT,__literal8,8byte_literals
        .align  3
LCPI0_0:
        .quad   4607632778762754458     ## double 1.1000000000000001
```
获取部分跟float类似, 只是存储到栈中占用了8 byte.
```
    movsd   %xmm0, -8(%rbp)
```

### 2.3 常量
字符常量/整数常量/浮点数常量在第2.2节中基本上已经介绍了其存储方式, 这里就不多做介绍.  
而`enum`枚举的的存储方式会有意思的多:
```
#include <stdio.h>

enum boolean {NO, YES};

main()
{
    int a = YES;
    int b = NO;
}
```
其汇编形式为:
```
        .section        __TEXT,__text,regular,pure_instructions
        .macosx_version_min 10, 12
        .globl  _main
        .align  4, 0x90
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
        movl    $1, -4(%rbp)
        movl    $0, -8(%rbp)
        popq    %rbp
        retq
        .cfi_endproc

.subsections_via_symbols
```
观察代码, 会发现里面找不到`enum boolean`任何相关的信息, 只能从复制到的变量a和b上下手:
```
        movl    $1, -4(%rbp)
        movl    $0, -8(%rbp)
```
从前面的介绍中的局部变量存放在栈的结论可以推论出, a存在`-4(%rbp)`位置而b存在`-8(%rbp)`位置, 对它们的赋值直接分别赋值的数值`1`和`0`, 也就是enum的本质就是一个int数值. 而也就意味着单从汇编代码上来看, 就比较难推测出某个enum数值所代表的符号意义(比如YES/NO).

### 2.4 声明
声明是用于告知编译器某个变量的存在以及其类型的, 变量在被实际赋值之前是不会占用任何空间的, 比如如下代码:
```
#include <stdio.h>

main()
{
    int lower, upper, step;
}
```
其汇编形式为:
```
        .section        __TEXT,__text,regular,pure_instructions
        .macosx_version_min 10, 12
        .globl  _main
        .align  4, 0x90
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
        popq    %rbp
        retq
        .cfi_endproc

.subsections_via_symbols
```
可以看出`_main`方法体里除了代码头和代码尾之外就没有任何其他东西了, 变量只有在做赋值时才会在栈上为其分配存储值的空间.

不过数组确实例外的, 在声明一个数组后, 数组的空间会直接在栈上被分配, 比如如下代码:
```
#include <stdio.h>

main()
{
    char line[1000];
}
```
其汇编形式为:
```
	.section	__TEXT,__text,regular,pure_instructions
	.macosx_version_min 10, 12
	.globl	_main
	.align	4, 0x90
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
	subq	$1008, %rsp             ## imm = 0x3F0
	movq	___stack_chk_guard@GOTPCREL(%rip), %rax
	movq	(%rax), %rcx
	movq	%rcx, -8(%rbp)
	movq	(%rax), %rax
	cmpq	-8(%rbp), %rax
	jne	LBB0_2
## BB#1:
	xorl	%eax, %eax
	addq	$1008, %rsp             ## imm = 0x3F0
	popq	%rbp
	retq
LBB0_2:
	callq	___stack_chk_fail
	.cfi_endproc


.subsections_via_symbols
```
从代码中可以看出栈被下移了1008 byte, 其中1000 byte用于存放`line[1000]`另外8 byte用来存放`___stack_chk_guard`.

### 2.5 算术操作符
常用的二元操作符`+, -, *, /, %`, 他们在使用时实际上对应了一条或者多条汇编指令. 先从`+`来看:
```
#include <stdio.h>

main()
{
    int a = 1;
    a = a + 2;
}
```
其汇编形式为:
```
	.section	__TEXT,__text,regular,pure_instructions
	.macosx_version_min 10, 12
	.globl	_main
	.align	4, 0x90
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
	xorl	%eax, %eax
	movl	$1, -4(%rbp)
	movl	-4(%rbp), %ecx
	addl	$2, %ecx
	movl	%ecx, -4(%rbp)
	popq	%rbp
	retq
	.cfi_endproc

.subsections_via_symbols
```
其关键代码是:
```
	movl	$1, -4(%rbp)
	movl	-4(%rbp), %ecx
	addl	$2, %ecx
	movl	%ecx, -4(%rbp)
```
把`1`存到栈顶偏移量`-4`位置分配给变量`a`的空间, 再把栈的变量`a`空间的内容取到寄存器`ecx`, 对`ecx`寄存做`addl`运算, 结果再存会栈上分配给`a`的空间.

看了`+`运算, `-`的运算也很雷同:
```
    subl	$2, %ecx    # 减
```
乘法`*`运算:
```
    imull   $3, -4(%rbp), %ecx
```
乘法`*`运算:
```
    imull   $3, -4(%rbp), %ecx
```
PS. 乘法运算有可能会被优化成向左位移预算, 比如'a = a * 2'相当于a左位移1位  
除法`/`运算:
```
	movl	$3, %ecx
	movl	$1, -4(%rbp)
	movl	-4(%rbp), %edx
	movl	%edx, %eax
	cltd
	idivl	%ecx
	movl	%eax, -4(%rbp)
```
除法运算会稍微复杂些, 会先把被除数放入`ecx`寄存器, 除数放入`eax`, 通过`idivl	%ecx`计算结果并把商存入`eax`, .

模`%`运算:
```
	movl	$3, %ecx
	movl	$1, -4(%rbp)
	movl	-4(%rbp), %edx
	movl	%edx, %eax
	cltd
	idivl	%ecx
	movl	%edx, -4(%rbp)
```
模运算的指令和除法运算一致, 差异就在于最后保存的结果不是`eax`里的商, 而是`edx`里面的余数.

PS. 浮点数的运算指令和整数不同哦, 大家可以自行尝试

### 2.6 比较运算符和逻辑运算符
比较运算符有`>, >=, <, <=, ==, !=`, 其中`>, >=, <, <=`的优先级相等, 而`==, !=`的优先级较低.  
所有比较运算符的指令都相同, 只是对比较的结果的处理会存在差异, 比如'>'的例子:
```
#include <stdio.h>

main()
{
    int a = 1;
    a = a > 1;
}
```
其汇编代码为:
```
        .section        __TEXT,__text,regular,pure_instructions
        .macosx_version_min 10, 12
        .globl  _main
        .align  4, 0x90
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
        movl    $1, -4(%rbp)
        cmpl    $1, -4(%rbp)
        setg    %cl
        andb    $1, %cl
        movzbl  %cl, %edx
        movl    %edx, -4(%rbp)
        popq    %rbp
        retq
        .cfi_endproc

.subsections_via_symbols
```
其中最关键的部分是这两句:
```
    cmpl    $1, -4(%rbp)
		setg    %cl
```
分别用于将`a`和`1`做比较, 然后如果比较的结果是`greater`则将%cl设为`1`, `setg`指令属于`SETcc`指令系列, 其中`cc`表示`condition code`也就是条件代码, 包括`E/NE/G/GE`(等于/不等于/大于/大于等于)等条件, 详情参见(x86 architecture condition codes)[http://www.sandpile.org/x86/cc.htm].

### 2.7 类型转换
在C代码中不同的数据类型之间可以进行有限的转换, 需要注意的是, 从`大`的数据类型转向`小`的数据类型可能导致信息的丢失.  
首先看看`int`和`char`类型之间的转换:
```
#include <stdio.h>

main()
{
    int a = 1;
    char c = 'a';
    a = c;
    c = a;
}
```
其汇编实现:
```
        .section        __TEXT,__text,regular,pure_instructions
        .macosx_version_min 10, 12
        .globl  _main
        .align  4, 0x90
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
        movl    $1, -4(%rbp)
        movb    $97, -5(%rbp)
        movsbl  -5(%rbp), %ecx
        movl    %ecx, -4(%rbp)
        movl    -4(%rbp), %ecx
        movb    %cl, %dl
        movb    %dl, -5(%rbp)
        popq    %rbp
        retq
        .cfi_endproc

.subsections_via_symbols
```
其中的关键部分是:
```
        movl    $1, -4(%rbp)
        movb    $97, -5(%rbp)
        movsbl  -5(%rbp), %ecx
        movl    %ecx, -4(%rbp)
        movl    -4(%rbp), %ecx
        movb    %cl, %dl
        movb    %dl, -5(%rbp)
```
栈上`-4(%rbp)`位置存的变量`a`内容是`1`, 大小是4 byte(根据指令`movl`来判断大小), 栈上`-5(%rbp)`位置存放的是变量`c`的内容`97`(也就是字符`a`), 大小是1 byte(根据指令`movb`来判断大小). `movsbl -5(%rbp), %ecx`将`-5(%rbp)`的内容从`b`(1字节)扩展为`l`(4字节)放入`ecx`, `movl %ecx, -4(%rbp)`将`ecx`的内容放入`a`变量的栈空间, 这两句完成了`a = c`语句.  
接下来的`movl -4(%rbp), %ecx`再将`a`变量的内容放回`ecx`, 后通过`movb %cl, %dl`将`ecx`的低8位`cl`放入`edx`的低8位`dl`, 将`dl`存入`c`变量的栈空间, 这两句完成了`c = a`语句. 值得关注的是`movb %cl, %dl`只取了`ecx`的低8位, 如果`ecx`的高位存在内容, 那么高位存的内容就被丢弃了.

再来看看`int`和`float`之间的转换:
```
#include <stdio.h>

main()
{
    int a;
    float c = 1.1;
    a = c;
    c = a;
}
```
其汇编形式为:
```
	.section	__TEXT,__text,regular,pure_instructions
	.macosx_version_min 10, 12
	.section	__TEXT,__literal4,4byte_literals
	.align	2
LCPI0_0:
	.long	1066192077              ## float 1.10000002
	.section	__TEXT,__text,regular,pure_instructions
	.globl	_main
	.align	4, 0x90
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
	xorl	%eax, %eax
	movss	LCPI0_0(%rip), %xmm0    ## xmm0 = mem[0],zero,zero,zero
	movss	%xmm0, -8(%rbp)
	cvttss2si	-8(%rbp), %ecx
	movl	%ecx, -4(%rbp)
	cvtsi2ssl	-4(%rbp), %xmm0
	movss	%xmm0, -8(%rbp)
	popq	%rbp
	retq
	.cfi_endproc

.subsections_via_symbols
```
其中关键部分是:
```
	movss	LCPI0_0(%rip), %xmm0    ## xmm0 = mem[0],zero,zero,zero
	movss	%xmm0, -8(%rbp)
	cvttss2si	-8(%rbp), %ecx
	movl	%ecx, -4(%rbp)
	cvtsi2ssl	-4(%rbp), %xmm0
	movss	%xmm0, -8(%rbp)
```
先将浮点数值从符号`LCPI0_0`位置放入`xmm0`寄存器, 再放入`-8(%rbp)`变量`c`的栈空间. `cvttss2si	-8(%rbp), %ecx`将变量`c`的值从浮点数转换为整数放入`ecx`, 之后将`ecx`放入`-4(%rbp)`变量`a`的栈空间, 完成`a = c`语句, `cvttss2si`将导致浮点数`1.1`转换为整数`1`丢失小数部分. 之后`cvtsi2ssl	-4(%rbp), %xmm0`将`a`的整数值转换为浮点数放入`xmm0`寄存器, 然后存入`c`的栈空间, 完成`c = a`语句.

### 2.8 自增自减操作
自增自减操作`++, --`的原理很简单大家应该都很明白, 而它们的汇编实现也非常简单, 有如下代码:
```
#include <stdio.h>

main()
{
   int i = 0;
   i++;
   i--;
}
```
其汇编实现为:
```
	.section	__TEXT,__text,regular,pure_instructions
	.macosx_version_min 10, 12
	.globl	_main
	.align	4, 0x90
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
	xorl	%eax, %eax
	movl	$0, -4(%rbp)
	movl	-4(%rbp), %ecx
	addl	$1, %ecx
	movl	%ecx, -4(%rbp)
	movl	-4(%rbp), %ecx
	addl	$-1, %ecx
	movl	%ecx, -4(%rbp)
	popq	%rbp
	retq
	.cfi_endproc

.subsections_via_symbols
```
关键代码为:
```
	movl	$0, -4(%rbp)
	movl	-4(%rbp), %ecx
	addl	$1, %ecx
	movl	%ecx, -4(%rbp)
	movl	-4(%rbp), %ecx
	addl	$-1, %ecx
	movl	%ecx, -4(%rbp)
```
先将`0`放入变量`i`的栈空间, 再将`i`的值放入`ecx`, `addl	$1, %ecx`实现`ecx = ecx + 1`, 再将`ecx`放回`i`的占空间, 完成`i++`操作. 而`i--`操作的区别在于`addl	$-1, %ecx`加的是`-1`, 实现`ecx = ecx - 1`.

### 2.9 位操作
位操作包括`&, | , ^, >>, <<, ~`(与,或,异或,右移,左移,非), 它们在汇编层面的实现和比较操作很类似.  
先看看与操作`&`:
```
#include <stdio.h>

main()
{
    int n = 1;
    n = n & 0177;    
}
```
其汇编实现如下:
```
	.section	__TEXT,__text,regular,pure_instructions
	.macosx_version_min 10, 12
	.globl	_main
	.align	4, 0x90
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
	xorl	%eax, %eax
	movl	$1, -4(%rbp)
	movl	-4(%rbp), %ecx
	andl	$127, %ecx
	movl	%ecx, -4(%rbp)
	popq	%rbp
	retq
	.cfi_endproc

.subsections_via_symbols
```
其中关键部分:
```
	movl	$1, -4(%rbp)
	movl	-4(%rbp), %ecx
	andl	$127, %ecx
	movl	%ecx, -4(%rbp)
```
首先将`1`存入变量`c`的栈空间, 然后将`c`的内容放入`ecx`, 通过`andl	$127, %ecx`将`ecx`的值与数值`127`做与运算, 最后将结果放回`c`的栈空间.

其他的位操作的实现都很相像, 只是使用的指令不同, 这里就不多做介绍, 大家可以自己尝试.

### 2.10 赋值操作和语句
赋值操作实际上在前面的例子里有了较多的说明, 对局部变量的赋值直接上是把运算的结果(一般存放在寄存器中), 保存到栈上分配给变量空间里. 有如下代码:
```
#include <stdio.h>

main()
{
    int i = 0;
    i = i + 2;
}
```
其汇编形式:
```
	.section	__TEXT,__text,regular,pure_instructions
	.macosx_version_min 10, 12
	.globl	_main
	.align	4, 0x90
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
	xorl	%eax, %eax
	movl	$0, -4(%rbp)
	movl	-4(%rbp), %ecx
	addl	$2, %ecx
	movl	%ecx, -4(%rbp)
	popq	%rbp
	retq
	.cfi_endproc

.subsections_via_symbols
```
关键代码为:
```
  movl	$0, -4(%rbp)
	movl	-4(%rbp), %ecx
	addl	$2, %ecx
	movl	%ecx, -4(%rbp)
```
把`0`放入变量`a`的栈空间, 再将`a`的内容放入`ecx`, 再计算`ecx = ecx + 2`, 最后再把`ecx`放回变量`a`的栈空间.

而如果我们把代码改为:
```
#include <stdio.h>

main()
{
    int i = 0;
    i += 2;
}
```
会发现其汇编代码:
```
	movl	$0, -4(%rbp)
	movl	-4(%rbp), %ecx
	addl	$2, %ecx
	movl	%ecx, -4(%rbp)
```
跟`i = i + 2`一模一样, 也就意味着`i += 2`可以算作C语言给我的一个让源代码更精简的语法糖! 它可以让类似`yyval[yypv[p3+p4] + yypv[p1]] = yyval[yypv[p3+p4] + yypv[p1]] + 2`这样的很长的句子变成更好读的`yyval[yypv[p3+p4] + yypv[p1]] += 2`.

前面的例子里讲述的都是变量之间的赋值, 那函数返回值的赋值呢? 有如下代码:
```
#include <stdio.h>

main()
{
    char a = getchar();
}
```
其汇编形式为:
```
	.section	__TEXT,__text,regular,pure_instructions
	.macosx_version_min 10, 12
	.globl	_main
	.align	4, 0x90
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
	subq	$16, %rsp
	callq	_getchar
	xorl	%ecx, %ecx
	movb	%al, %dl
	movb	%dl, -1(%rbp)
	movl	%ecx, %eax
	addq	$16, %rsp
	popq	%rbp
	retq
	.cfi_endproc

.subsections_via_symbols
```
其关键代码为:
```
	callq	_getchar
	movb	%al, %dl
	movb	%dl, -1(%rbp)
```
首先通过`callq	_getchar`调用方法, 再将寄存器`al`内容放入寄存器`dl`, 注意, 方法调用返回值被放入了`al`中! 这里的`_getchar`返回值是`char`占空间1 byte, 所以被放入了寄存器`eax`的低8位`al`中, 如果方法的返回值是`int`类型占4 byte, 就会被放入寄存器`eax`中!

### 2.11 条件语句
有如下代码:
```
#include <stdio.h>

main()
{
    int z, a = 2, b = 1;
    if (a > b) {
        z = a;
    } else {
        z = b;
    }
}
```
其汇编形式如下:
```
	.section	__TEXT,__text,regular,pure_instructions
	.macosx_version_min 10, 12
	.globl	_main
	.align	4, 0x90
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
	movl	$0, -4(%rbp)
	movl	$2, -12(%rbp)
	movl	$1, -16(%rbp)
	movl	-12(%rbp), %eax
	cmpl	-16(%rbp), %eax
	jle	LBB0_2
## BB#1:
	movl	-12(%rbp), %eax
	movl	%eax, -8(%rbp)
	jmp	LBB0_3
LBB0_2:
	movl	-16(%rbp), %eax
	movl	%eax, -8(%rbp)
LBB0_3:
	movl	-4(%rbp), %eax
	popq	%rbp
	retq
	.cfi_endproc

.subsections_via_symbols
```
关键部分为:
```
	movl	-12(%rbp), %eax
	cmpl	-16(%rbp), %eax
	jle	LBB0_2
## BB#1:
	movl	-12(%rbp), %eax
	movl	%eax, -8(%rbp)
	jmp	LBB0_3
LBB0_2:
	movl	-16(%rbp), %eax
	movl	%eax, -8(%rbp)
```
将变量`a`的值放入`eax`, 将`eax`的值与变量`c`的值做比较, `jle	LBB0_2`判断比较的结果如果是`le`(小于等于)则跳到标签`LBB0_2`处执行, 如果比较结果是大于则顺序往下执行. 从这里可以看出实际上`if`语句就是一个`cmp`比较语句加上`j`跳转语句来根据比较结果跳转到不同的代码分支. 在第一章例子中提到过的`for`语句的实现类似.

### 2.12 优先级和顺序
C语言中操作符是有顺序和优先级的, 具体的内容可以翻阅原版书籍, 这里只做简单的举例对优先级的实现进行说明.  
有如下代码:
```
#include <stdio.h>

main()
{
    int a,b,c;
    a = a | b & c;
}
```
代码在编译时编译器clang会给出提示`place parentheses around the '&' expression to silence this warning`, 原因是`&`具有比`|`更高的优先级, 不加括号导致代码容易被误解.  
其汇编实现如下:
```
	.section	__TEXT,__text,regular,pure_instructions
	.macosx_version_min 10, 12
	.globl	_main
	.align	4, 0x90
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
	xorl	%eax, %eax
	movl	-4(%rbp), %ecx
	movl	-8(%rbp), %edx
	andl	-12(%rbp), %edx
	orl	%edx, %ecx
	movl	%ecx, -4(%rbp)
	popq	%rbp
	retq
	.cfi_endproc

.subsections_via_symbols
```
关键部分代码:
```
	movl	-4(%rbp), %ecx
	movl	-8(%rbp), %edx
	andl	-12(%rbp), %edx
	orl	%edx, %ecx
	movl	%ecx, -4(%rbp)
```
先把量量`a`(`-4(%rbp)`),`b`(`-8(%rbp)`)的值放入`ecx`,`edx`, 再将变量`c`(`-12(%rbp)`)的值与`edx`做与运算, 再将`ecx`与`edx`做或运算, 结果存回`a`的栈空间. 从汇编代码可以看出, `and`操作在`or`之前, 也就是`&`比`|`具有更高的优先级.

细心的大家应该已经发现,源代码中的`int a,b,c;`只对变量做了声明而没有做初始化, 从汇编代码中也可以看出, 只为变量分配了空间而没有给空间里面放入默认值, 也就意味着如果为变量分配空间里原本有值, 那么变量的值就是空间里的值, 这样变量的初始值就未知了. 值得一提的是, 对栈的分配/释放操作只会对栈指针做加减法, 而不会对栈内存中的内容做任何修改(也不会把释放的栈空间设置为0).
第一章 一些实例
===

多说无用, 直接从实例开始讲吧!

### 1.1 开始
讲语言的第一个例子自然是在控制台打印:
```
hello, world
```
想必大家都可以很轻易的用C写出如下代码:
```
#include <stdio.h>

main()
{
    printf("hello, world\n");
}
```
将上述代码保存至`helloworld.c`, 并使用clang进行编译:
```
clang helloworld.c
```
得到`a.out`产物, 在命令行执行它, 将打印出如下内容:
```
hello, world
```
咳咳, 这并不是想要的结果, 还是祭出clang汇编吧:
```
clang -S helloworld.c
```
该指令会生成同名文件`helloworld.s`, 其内容如下:
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
	leaq	L_.str(%rip), %rdi
	movb	$0, %al
	callq	_printf
	xorl	%ecx, %ecx
	movl	%eax, -4(%rbp)          ## 4-byte Spill
	movl	%ecx, %eax
	addq	$16, %rsp
	popq	%rbp
	retq
	.cfi_endproc

	.section	__TEXT,__cstring,cstring_literals
L_.str:                                 ## @.str
	.asciz	"hello, world\n"


.subsections_via_symbols
```
代码中类似`.section`或`.globl`等以`.`开头的, 被称之为编译器指令, 用于告知编译器相关的信息或者进行特定操作.

类似`_main:`或`Ltmp0:`的, 被称之为标签(label), 用于辅助定位代码或者资源地址, 也便于开发者理解和记忆.

类似`pushq`或`movq`的, 被称之为汇编指令, 它们会被汇编器编译为机器代码, 最终被cpu所执行.

上述代码中, 用`.section`指令生成了两个段:
* `__TEXT,__text`用来存放代码指令, 代码一般都放在这一节.
* `__TEXT,__cstring`用来存放`c string`也就是字符串, 可以看到`hello, world\n`字符串就存放在这里.


`hello, world\n`字符串前一行是一个标签(label)`L_.str`, 在之前的代码中`leaq	L_.str(%rip), %rdi`指令引用了`L_.str`这个标签, 在经过汇编器汇编后会将标汇编为字符串所存放的地址, 让程序可以定位到字符串.

指令`callq	_printf`将`%rdi`作为第一个参数(里面存放的是"hello, world\n"字符串的地址)调用`_printf`方法.

代码中的如下部分被称之为方法头(prologue), 用于保存上一个方法调用栈帧的帧头及预留部分栈空间用于局部变量:
```
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
```

代码中如下部分被称之为方法尾(epilogue), 用于取出方法头中栈帧信息及方法的返回地址, 并将栈恢复到方法调用前的位置:
```
	xorl	%ecx, %ecx
	movl	%eax, -4(%rbp)          ## 4-byte Spill
	movl	%ecx, %eax
	addq	$16, %rsp
	popq	%rbp
	retq
```
调用栈的栈帧(stack frame)可以用来回溯调用栈，可以用于生成调用栈。

### 1.2 变量和算术表达式
下一段程序用于输出华氏温度和摄氏温度的值的对应关系, 其运算公式为`C=(5/9)(F-32)`, 程序输出如下结果:
```
0	-17
20	-6
40	4
60	15
80	26
100	37
120	48
140	60
160	71
180	82
200	93
220	104
240	115
260	126
280	137
300	148
```
对应的代码如下:
```
#include <stdio.h>

/* print Fahrenheit-Celsius table for fahr = 0, 20, ..., 300 */ 
main() {
    int fahr, celsius;
    int lower, upper, step;

    lower = 0;    /* lower limit of temperature scale */
    upper = 300;  /* upper limit */
    step = 20;     /* step size */

    fahr = lower;
    while (fahr <= upper) {
        celsius = 5 * (fahr-32) / 9;
        printf("%d\t%d\n", fahr, celsius);
        fahr = fahr + step;
    }
}
```
这段代码输出的汇编如下:
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
        subq    $32, %rsp
        movl    $0, -4(%rbp)
        movl    $0, -16(%rbp)
        movl    $300, -20(%rbp)         ## imm = 0x12C
        movl    $20, -24(%rbp)
        movl    -16(%rbp), %eax
        movl    %eax, -8(%rbp)
LBB0_1:                                 ## =>This Inner Loop Header: Depth=1
        movl    -8(%rbp), %eax
        cmpl    -20(%rbp), %eax
        jg      LBB0_3
## BB#2:                                ##   in Loop: Header=BB0_1 Depth=1
        leaq    L_.str(%rip), %rdi
        movl    $9, %eax
        movl    -8(%rbp), %ecx
        subl    $32, %ecx
        imull   $5, %ecx, %ecx
        movl    %eax, -28(%rbp)         ## 4-byte Spill
        movl    %ecx, %eax
        cltd
        movl    -28(%rbp), %ecx         ## 4-byte Reload
        idivl   %ecx
        movl    %eax, -12(%rbp)
        movl    -8(%rbp), %esi
        movl    -12(%rbp), %eax
        movl    %eax, %edx
        movb    $0, %al
        callq   _printf
        movl    -8(%rbp), %ecx
        addl    -24(%rbp), %ecx
        movl    %ecx, -8(%rbp)
        movl    %eax, -32(%rbp)         ## 4-byte Spill
        jmp     LBB0_1
LBB0_3:
        movl    -4(%rbp), %eax
        addq    $32, %rsp
        popq    %rbp
        retq
        .cfi_endproc

        .section        __TEXT,__cstring,cstring_literals
L_.str:                                 ## @.str
        .asciz  "%d\t%d\n"


.subsections_via_symbols
```
这里先说重点：局部变量是放在栈上的！  
其中的关键点和解释如下:
```
        movq    %rsp, %rbp
Ltmp2:
        .cfi_def_cfa_register %rbp
        subq    $32, %rsp
        movl    $0, -4(%rbp)
        movl    $0, -16(%rbp)
        movl    $300, -20(%rbp)         ## imm = 0x12C
        movl    $20, -24(%rbp)
```
其中`movq %rsp, %rbp`, 将`%rsp`复制到`%rbp`, 而`%rsp`内存放的是栈的地址, 此时`rsp`和`rbp`都指向栈低. `subq $32, %rsp`, 也就是`rsp = rsp - 32`将栈地址减去32, 用于存放局部变量. `movl $0, -4(%rbp)`将`0`存放于`rbp - 4`的值作为指针指向的内存地址, 也就是栈顶下方4 byte位置, 从后面的代码中推测出这里存放的是`main`方法的默认返回值`0`. `movl $0, -16(%rbp)`, 也就是栈顶下方16 byte位置存放的是`lower`变量的初始值`0`. 同理, 后两句分别存放的是`upper`和`step`变量的值.

再接下来的两句就有点儿意思:
```
        movl    -16(%rbp), %eax
        movl    %eax, -8(%rbp)
```
`movl -16(%rbp), %eax`, 将`lower`的值取出放到`eax`中. `movl %eax, -8(%rbp)`, 再将`eax`的值存放于栈顶下8 byte位置, 这里也就是给`fahr`赋值了(`fahr = lower;` 这句).

后面就开始进入循环了:
```
        movl    -8(%rbp), %eax
        cmpl    -20(%rbp), %eax
        jg      LBB0_3
```
`movl -8(%rbp), %eax`将`fahr`(理解下为什么是fahr)放入`eax`. `cmpl -20(%rbp), %eax`将`fahr`和`upper`做比较. `jg LBB0_3`如果比较结果是大于, 则调转到标签`LBB0_3`也就是方法尾部, 这里`<=`被转换为了`jg`(大于则跳转).

再往下看到接近末尾的`jmp LBB0_1`指令, 它会跳转回标签`LBB0_1`处, 将重新赋值后的`fahr`和`upper`做比较, 来判断是否跳出循环.

其他的操作的概念在前面基本上在前面都已经涵盖到, 对应的汇编指令都可以在网上查到详细的资料, 这里就不多做复述.

### 1.3 for循环
将上一节的代码用for循环来实现:
```
#include <stdio.h>

/* print Fahrenheit-Celsius table */ 
main() {
    int fahr;
		
    for (fahr = 0; fahr <= 300; fahr = fahr + 20)
		    printf("%3d %6.1f\n", fahr, (5.0/9.0)*(fahr-32));
}
```
其汇编代码如下:
```
        .section        __TEXT,__text,regular,pure_instructions
        .macosx_version_min 10, 12
        .section        __TEXT,__literal8,8byte_literals
        .align  3
LCPI0_0:
        .quad   4603179219131243634     ## double 0.55555555555555558
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
        subq    $16, %rsp
        movl    $0, -4(%rbp)
        movl    $0, -8(%rbp)
LBB0_1:                                 ## =>This Inner Loop Header: Depth=1
        cmpl    $300, -8(%rbp)          ## imm = 0x12C
        jg      LBB0_4
## BB#2:                                ##   in Loop: Header=BB0_1 Depth=1
        leaq    L_.str(%rip), %rdi
        movsd   LCPI0_0(%rip), %xmm0    ## xmm0 = mem[0],zero
        movl    -8(%rbp), %esi
        movl    -8(%rbp), %eax
        subl    $32, %eax
        cvtsi2sdl       %eax, %xmm1
        mulsd   %xmm1, %xmm0
        movb    $1, %al
        callq   _printf
        movl    %eax, -12(%rbp)         ## 4-byte Spill
## BB#3:                                ##   in Loop: Header=BB0_1 Depth=1
        movl    -8(%rbp), %eax
        addl    $20, %eax
        movl    %eax, -8(%rbp)
        jmp     LBB0_1
LBB0_4:
        movl    -4(%rbp), %eax
        addq    $16, %rsp
        popq    %rbp
        retq
        .cfi_endproc

        .section        __TEXT,__cstring,cstring_literals
L_.str:                                 ## @.str
        .asciz  "%3d %6.1f\n"


.subsections_via_symbols
```

源代码中有一个常数运算`5.0/9.0`, 被编译器给优化成了这样:
```
LCPI0_0:
        .quad   4603179219131243634     ## double 0.55555555555555558
```

for循环的头部判断是否结束循环, 同样用的`cmpl`指令和`jg`指令:
```
LBB0_1:                                 ## =>This Inner Loop Header: Depth=1
        cmpl    $300, -8(%rbp)          ## imm = 0x12C
        jg      LBB0_4
```
而对变量`fahr`的加`20`的操作实际上在循环的尾部来做的:
```
## BB#3:                                ##   in Loop: Header=BB0_1 Depth=1
        movl    -8(%rbp), %eax
        addl    $20, %eax
        movl    %eax, -8(%rbp)
        jmp     LBB0_1
```
上述代码将`fahr`从栈里取到寄存器, 加上`20`, 塞回栈里, 跳回循环头部做判断.

### 1.4 符号常量
直接上代码:
```
#include <stdio.h>

#define LOWER 0    /* lower limit of table */
#define UPPER 300  /* upper limit */
#define STEP  20   /* step size */

/* print Fahrenheit-Celsius table */
main() {
    int fahr;
		
    for (fahr = LOWER; fahr <= UPPER; fahr = fahr + STEP)
        printf("%3d %6.1f\n", fahr, (5.0/9.0)*(fahr-32));
}
```
以上代码保存为`f2cforpreprocessor.c`, 经过编译器的预处理(preprocessor):
```
clang -E f2cforpreprocessor.c
```
生成
```
.....省略头文件的预处理部分.....
main() {
    int fahr;
		
    for (fahr = 0; fahr <= 300; fahr = fahr + 20)
        printf("%3d %6.1f\n", fahr, (5.0/9.0)*(fahr-32));
}
```
代码本身没有变化, 这里就不再重复介绍.

### 1.5 数组
原文对数组简介的代码较长, 为了避免重复, 这里单独对数组部分做介绍. 先上代码:
```
#include <stdio.h>

main()
{
    int ndigit[10];
}
```
其汇编代码如下:
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
        subq    $48, %rsp
        movq    ___stack_chk_guard@GOTPCREL(%rip), %rax
        movq    (%rax), %rcx
        movq    %rcx, -8(%rbp)
        movq    (%rax), %rax
        cmpq    -8(%rbp), %rax
        jne     LBB0_2
## BB#1:
        xorl    %eax, %eax
        addq    $48, %rsp
        popq    %rbp
        retq
LBB0_2:
        callq   ___stack_chk_fail
        .cfi_endproc


.subsections_via_symbols
```
其中最核心的代码是这一部分;
```
Ltmp2:
        .cfi_def_cfa_register %rbp
        subq    $48, %rsp
        movq    ___stack_chk_guard@GOTPCREL(%rip), %rax
        movq    (%rax), %rcx
        movq    %rcx, -8(%rbp)
        movq    (%rax), %rax
        cmpq    -8(%rbp), %rax
        jne     LBB0_2
```
在x86_64架构下面`int`的长度是4个byte, 那么拥有10个元素数组的`ndigit`所占用的空间是40个byte, 但`subq $48, %rsp`却把栈指针`rsp`减去了48, 那多出来的8个byte里面存放的什么呢?是`___stack_chk_guard`, 它是一个外部的符号, 里面存放的是一个数值, 将它放在数组的最后一个元素之后, 如果数组被越界写入(栈被破坏), 后面的`jne LBB0_2`将跳转到`callq ___stack_chk_fail`栈校验失败处理方法.

### 1.6 方法
前面的小节中讨论的内容都是语句级别的, 那么方法的表现形式是什么呢? 先看代码:
```
#include <stdio.h>

int power(int m, int n); /* test power function */

int main() {
    return power(2,1);
}

int power(int base, int n) {
    return base;
}
```
该代码的汇编代码如下:
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
        subq    $16, %rsp
        movl    $2, %edi
        movl    $1, %esi
        movl    $0, -4(%rbp)
        callq   _power
        addq    $16, %rsp
        popq    %rbp
        retq
        .cfi_endproc

        .globl  _power
        .align  4, 0x90
_power:                                 ## @power
        .cfi_startproc
## BB#0:
        pushq   %rbp
Ltmp3:
        .cfi_def_cfa_offset 16
Ltmp4:
        .cfi_offset %rbp, -16
        movq    %rsp, %rbp
Ltmp5:
        .cfi_def_cfa_register %rbp
        movl    %edi, -4(%rbp)
        movl    %esi, -8(%rbp)
        movl    -4(%rbp), %eax
        popq    %rbp
        retq
        .cfi_endproc


.subsections_via_symbols
```
对比之前节中的汇编代码, 会发现在源代码中新增加了`power`方法后, 汇编代码中多出了如下内容:
```
        .globl  _power
        .align  4, 0x90
_power:                                 ## @power
        .cfi_startproc
```
这段跟`_main`方法类似, 其中`.globl _power`告知编译器`_power`符号全局可见. 其实对于汇编来说, 方法就是一个在代码段的有名字的地址.

### 1.7 参数 - 方法调用传值
直接用节1.6中的汇编代码来做介绍, 在`main`方法中调用`power`的时候传递了两个参数`2`和`1`, 其汇编实现如下:
```
        movl    $2, %edi
        movl    $1, %esi
        movl    $0, -4(%rbp)
        callq   _power
```
其中`movl $2, %edi`将`2`放在`edi`中作为第一个参数, `movl $1, %esi`将`1`放在`esi`中作为第二个参数, `movl $0, -4(%rbp)`这句没用到, `callq _power`调用`_power`方法.

决定第几个参数放在什么寄存器里面参见llvm的代码[X86CallingConv.td](https://github.com/llvm-mirror/llvm/blob/master/lib/Target/X86/X86CallingConv.td)：
```
  // The first 6 integer arguments are passed in integer registers.
  CCIfType<[i32], CCAssignToReg<[EDI, ESI, EDX, ECX, R8D, R9D]>>,
  CCIfType<[i64], CCAssignToReg<[RDI, RSI, RDX, RCX, R8 , R9 ]>>,
```
_注：不同编译器可能实现不同，不同平台的实现可能不同_  
代码提示前6个整数参数用整数寄存器来传，而更多的可以参数放在栈上传(见intel手册6.3.3)。

### 1.8 外部变量和作用域
分别有两个文件`extern1.c`内容如下:
```
#include <stdio.h>

void copy(void);
char line[1000];
int main() {
    line[0] = 'a';
    copy();
}
```
和`extern2.c`内容如下:
```
void copy()
{
    extern char line[];
    line[1] = line[0];
}
```
他们均能被单独编译为汇编文件`extern1.s`:
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
	movq	_line@GOTPCREL(%rip), %rax
	movb	$97, (%rax)
	callq	_copy
	xorl	%eax, %eax
	popq	%rbp
	retq
	.cfi_endproc

	.comm	_line,1000,4            ## @line

.subsections_via_symbols
```
及`extern2.s`:
```
	.section	__TEXT,__text,regular,pure_instructions
	.macosx_version_min 10, 12
	.globl	_copy
	.align	4, 0x90
_copy:                                  ## @copy
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
	movq	_line@GOTPCREL(%rip), %rax
	movb	(%rax), %cl
	movb	%cl, 1(%rax)
	popq	%rbp
	retq
	.cfi_endproc


.subsections_via_symbols
```
在`extern1.s`中`.comm	_line,1000,4`会为`_line`符号在全局`__DATA,__common`节中分配一段空间, 使得在`extern2.s`中被`movq	_line@GOTPCREL(%rip), %rax`找到并访问.

而在`extern2.s`中`.globl	_copy`会将`_copy`符号暴露在全局, 使得在`extern1.s`中被`callq	_copy`访问到.

各符号所对应的具体地址会在链接(link)的过程中被确定和关联. 执行指令`clang extern1.c extern2.c`编译出产物`a.out`, 通过otool工具可以查看编译后的产物的信息, 如`otool -tV a.out`可以查看`__TEXT`段的反汇编:
```
a.out:
(__TEXT,__text) section
_main:
0000000100000f80	pushq	%rbp
0000000100000f81	movq	%rsp, %rbp
0000000100000f84	leaq	_line(%rip), %rax
0000000100000f8b	movb	$0x61, (%rax)
0000000100000f8e	callq	_copy
0000000100000f93	xorl	%eax, %eax
0000000100000f95	popq	%rbp
0000000100000f96	retq
0000000100000f97	nop
0000000100000f98	nop
0000000100000f99	nop
0000000100000f9a	nop
0000000100000f9b	nop
0000000100000f9c	nop
0000000100000f9d	nop
0000000100000f9e	nop
0000000100000f9f	nop
_copy:
0000000100000fa0	pushq	%rbp
0000000100000fa1	movq	%rsp, %rbp
0000000100000fa4	leaq	_line(%rip), %rax
0000000100000fab	movb	(%rax), %cl
0000000100000fad	movb	%cl, 0x1(%rax)
0000000100000fb0	popq	%rbp
0000000100000fb1	retq
```
其中可以看到`0000000100000f84	leaq	_line(%rip), %rax`和`0000000100000fa4	leaq	_line(%rip), %rax`对`_line`符号进行了引用. 而`line`的地址可以通过`nm a.out`查看:
```
0000000100000000 T __mh_execute_header
0000000100000fa0 T _copy
0000000100001000 S _line
0000000100000f80 T _main
                 U dyld_stub_binder
```
可以看出`_line`位于地址`0000000100001000`处.
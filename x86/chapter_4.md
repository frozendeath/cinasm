第四章 函数和程序结构
===

### 4.1 函数基础
先来一个最简单的函数, 其参数和返回值均为整数`int`:
```
#include <stdio.h>

int add(int a, int b) {
    return a+b;
}
```
其汇编实现如下:
```
_add:                                   ## @add
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
	movl	%edi, -4(%rbp)     # 将寄存器edi存入栈-4位置, 参数a
	movl	%esi, -8(%rbp)     # 将寄存器edi存入栈-4位置, 参数b
	movl	-4(%rbp), %esi     # 参数a捞出来放入寄存器esi
	addl	-8(%rbp), %esi     # 参数b和寄存器esi值相加放入esi
	movl	%esi, %eax         # 寄存器esi值放入寄存器eax作为返回值
	popq	%rbp
	retq
	.cfi_endproc
```
从代码中我们可以推测, 寄存器`edi`在函数调用时用于存放第一个整数参数, 寄存器`esi`用于存放第二个整数参数, 寄存器`eax`中存储函数的返回值. 真实情况是这样么? 我们在main方法中调用一下`add`方法试试:
```
#include <stdio.h>

int add(int a, int b) {
    return a+b;
}

int main(int argc, const char *argv[]) {
    return add(1,2) + 1;
}
```
`main`方法的汇编实现如下:
```
_main:                                  ## @main
	.cfi_startproc
## BB#0:
	pushq	%rbp
Ltmp3:
	.cfi_def_cfa_offset 16
Ltmp4:
	.cfi_offset %rbp, -16
	movq	%rsp, %rbp
Ltmp5:
	.cfi_def_cfa_register %rbp
	subq	$16, %rsp
	movl	$1, %eax                 # 数字1放入寄存器eax 
	movl	$2, %ecx                 # 数字2放入寄存器ecx 
	movl	$0, -4(%rbp)
	movl	%edi, -8(%rbp)
	movq	%rsi, -16(%rbp)
	movl	%eax, %edi               # 寄存器eax的值放入edi, 也就是1放入了edi作为第一个参数, 符合推测
	movl	%ecx, %esi               # 寄存器ecx的值放入esi, 也就是2放入了esi作为第二个参数, 符合推测
	callq	_add
	addl	$1, %eax                 # 将数字1与寄存器eax值相加放入eax, 也就是1与add函数的返回值相加, 符合推测
	addq	$16, %rsp
	popq	%rbp
	retq
	.cfi_endproc
```

如果有更多的参数怎么传递呢? 来个有10个参数的例子:
```
#include <stdio.h>

int add(int a, int b, int c, int d, int e, int f, int g, int h, int i, int j) {
    return a+b+c+d+e+f+g+h+i+j;
}

int main(int argc, const char *argv[]) {
    return add(1,2,3,4,5,6,7,8,9,10);
}
```
其汇编实现为:
```
_main:                                  ## @main
	.cfi_startproc
## BB#0:
	pushq	%rbp
Ltmp4:
	.cfi_def_cfa_offset 16
Ltmp5:
	.cfi_offset %rbp, -16
	movq	%rsp, %rbp
Ltmp6:
	.cfi_def_cfa_register %rbp
	pushq	%r15
	pushq	%r14
	pushq	%rbx
	subq	$72, %rsp
Ltmp7:
	.cfi_offset %rbx, -40
Ltmp8:
	.cfi_offset %r14, -32
Ltmp9:
	.cfi_offset %r15, -24
	movl	$1, %eax
	movl	$2, %ecx
	movl	$3, %edx                # 参数c           
	movl	$4, %r8d
	movl	$5, %r9d
	movl	$6, %r10d
	movl	$7, %r11d
	movl	$8, %ebx
	movl	$9, %r14d
	movl	$10, %r15d
	movl	$0, -28(%rbp)
	movl	%edi, -32(%rbp)
	movq	%rsi, -40(%rbp)
	movl	%eax, %edi              # 参数a
	movl	%ecx, %esi              # 参数b
	movl	%r8d, %ecx              # 参数d
	movl	%r9d, %r8d              # 参数e
	movl	%r10d, %r9d             # 参数f
	movl	$7, (%rsp)              # 参数g
	movl	$8, 8(%rsp)             # 参数h
	movl	$9, 16(%rsp)            # 参数i
	movl	$10, 24(%rsp)           # 参数j
	movl	%r15d, -44(%rbp)        ## 4-byte Spill
	movl	%r14d, -48(%rbp)        ## 4-byte Spill
	movl	%ebx, -52(%rbp)         ## 4-byte Spill
	movl	%r11d, -56(%rbp)        ## 4-byte Spill
	callq	_add
	addq	$72, %rsp
	popq	%rbx
	popq	%r14
	popq	%r15
	popq	%rbp
	retq
	.cfi_endproc
```
也就是`edi`, `esi`, `edx`, `ecx`, `r8d`, `r9d`中顺序存第1~6个参数, 后面的参数存在`(%rsp)`, `8(%rsp)`, `16(%rsp)`, `24(%rsp)`, 也就是栈上!

### 4.2 浮点数参数和返回值的函数
刚刚的示例中的参数和返回值类型都是整数, 那浮点数的情形如何呢? 直接上有10个参数的例子:
```
#include <stdio.h>

double add(double a, double b, double c, double d, double e, double f, double g, double h, double i, double j) {
    return a+b+c+d+e+f+g+h+i+j;
}

int main(int argc, const char *argv[]) {
    return add(1.0,2.0,3.0,4.0,5.0,6.0,7.0,8.0,9.0,10.0);
}
```
其汇编形式如下:
```
	.section	__TEXT,__literal8,8byte_literals
	.p2align	3
LCPI1_0:
	.quad	4607182418800017408     ## double 1
LCPI1_1:
	.quad	4611686018427387904     ## double 2
LCPI1_2:
	.quad	4613937818241073152     ## double 3
LCPI1_3:
	.quad	4616189618054758400     ## double 4
LCPI1_4:
	.quad	4617315517961601024     ## double 5
LCPI1_5:
	.quad	4618441417868443648     ## double 6
LCPI1_6:
	.quad	4619567317775286272     ## double 7
LCPI1_7:
	.quad	4620693217682128896     ## double 8
LCPI1_8:
	.quad	4621256167635550208     ## double 9
LCPI1_9:
	.quad	4621819117588971520     ## double 10
	.section	__TEXT,__text,regular,pure_instructions
	.globl	_main
	.p2align	4, 0x90
_main:                                  ## @main
	.cfi_startproc
## BB#0:
	pushq	%rbp
Ltmp3:
	.cfi_def_cfa_offset 16
Ltmp4:
	.cfi_offset %rbp, -16
	movq	%rsp, %rbp
Ltmp5:
	.cfi_def_cfa_register %rbp
	subq	$32, %rsp
	movsd	LCPI1_0(%rip), %xmm0    ## xmm0 = mem[0],zero
	movsd	LCPI1_1(%rip), %xmm1    ## xmm1 = mem[0],zero
	movsd	LCPI1_2(%rip), %xmm2    ## xmm2 = mem[0],zero
	movsd	LCPI1_3(%rip), %xmm3    ## xmm3 = mem[0],zero
	movsd	LCPI1_4(%rip), %xmm4    ## xmm4 = mem[0],zero
	movsd	LCPI1_5(%rip), %xmm5    ## xmm5 = mem[0],zero
	movsd	LCPI1_6(%rip), %xmm6    ## xmm6 = mem[0],zero
	movsd	LCPI1_7(%rip), %xmm7    ## xmm7 = mem[0],zero
	movsd	LCPI1_8(%rip), %xmm8    ## xmm8 = mem[0],zero
	movsd	LCPI1_9(%rip), %xmm9    ## xmm9 = mem[0],zero
	movl	$0, -4(%rbp)
	movl	%edi, -8(%rbp)
	movq	%rsi, -16(%rbp)
	movsd	%xmm8, (%rsp)           # 9.0 放到了栈里
	movsd	%xmm9, 8(%rsp)          # 10.0 放到了栈里
	callq	_add
	cvttsd2si	%xmm0, %eax         # 返回值在寄存器xmm0
	addq	$32, %rsp
	popq	%rbp
	retq
	.cfi_endproc
```
从汇编代码看来, 浮点数的参数传递比整数容易理解多了, 从寄存器xmm0~xmm7分别存放了第1~8个参数, 第9~10个参数放到了`(%rsp)`和`8(%rsp)`也就是栈里, 而返回值则存在寄存器xmm0中.

### 4.3 外部变量
这里的外部变量包含变量和函数, 对于汇编层面来讲, 外部的变量和函数其实都是一个可被找到的符号(更确切的来说应该是地址). 例如如下代码:
```
#include <stdio.h>

int a = 1;

int main()
{
    return a++;
}
```
其汇编实现为:
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
	movl	$0, -4(%rbp)
	movl	_a(%rip), %eax          # 将符号a的内容放入寄存器eax
	movl	%eax, %ecx
	addl	$1, %ecx
	movl	%ecx, _a(%rip)          # 再将a++的结果放回a
	popq	%rbp
	retq
	.cfi_endproc

	.section	__DATA,__data
	.globl	_a                      ## @a
	.p2align	2
_a:                                 # 符号a
	.long	1                       ## 0x1
```
其中`_a(%rip)`中的`_a`代表符号a相对于`rip`的偏移也就是相对地址(相对于rip), 就像`8(%rsp)`是相对于栈的偏移一样, 而`_a(%rip)`整个写法表示的是符号`a`的绝对地址.

而对于外部函数的调用也一样, 第4.1节中有`callq	_add`指令, 其中`_add`代表的是`add`函数相对于`rip`的相对地址(callq指令隐去了对rip偏移的表示).

### 4.4 作用域
对于汇编来说, 其实并不太有作用域的概念, 所有的数据都存在栈/符号/寄存器中, 在汇编层面对于数据的操作具有绝对的访问/控制能力. 这里就讲下原书中例子提到的`extern`:
```
#include <stdio.h>

extern int a;

int main()
{
    return a++;
}
```
其汇编实现如下
```
	.section	__TEXT,__text,regular,pure_instructions
	.macosx_version_min 10, 12
	.globl	_main
	.p2align	4, 0x90
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
	movq	_a@GOTPCREL(%rip), %rax    # 从GOTPCREL表中取到a符号的偏移量
	movl	$0, -4(%rbp)
	movl	(%rax), %ecx
	movl	%ecx, %edx
	addl	$1, %edx
	movl	%edx, (%rax)
	movl	%ecx, %eax
	popq	%rbp
	retq
	.cfi_endproc
	
.subsections_via_symbols
```
在汇编代码中找不到对`a`符号的直接定义, 而是通过`_a@GOTPCREL(%rip)`方式来从全局的`GOTPCREL`中找到`a`符号的地址计算偏移量, 这里`GOTPCREL`的意思是`Global Offset Table PC Relative`. 在链接的过程中, 会将其他源代码中的`a`符号放入`GOTPCREL`表, 让`a`符号可以被全局访问到.

### 4.5 头文件
在编译时，如果一个.c源码文件include了一个.h头文件，这个.h文件内的文件的所有内容会被直接替换调原有的include指令。比如：  
有头文件
```
// a.h
void a();
void b();
```
和源代码
```
// a.c
#include "a.h"

void c() {
    a();
	b();
}
```
在编译时会生成临时的源代码(示意):
```
// a.c
// a.h
void a();
void b();

void c() {
    a();
	b();
}
```

头文件的主要意义在于让编译器可以找到特定方法/变量/数据结构（统称为符号）的申明，使得在只有库没有没有源代码情况下，确认某些符号是否存在，如果头文件中没有具体的实现/定义, 将对汇编代码的生成没有影响. 很多新的高级语言比如swift/go/java都已经移除了头文件的概念.

### 4.6 静态变量
静态变量(static)分为两种情况, 全局静态变量和内部静态变量.  
全局静态变量是指放在方法体外的变量前加上static修饰, 它使得全局变量对外不可见. 比如:
```
#include <stdio.h>

static int a = 1;
void empty() {
    a++;
}
```
和
```
#include <stdio.h>

int a = 1;
void empty() {
    a++;
}
```
的汇编实现:
```
	.section	__DATA,__data
	.p2align	2               ## @a
_a:
	.long	1                       ## 0x1
```
和:
```
	.section	__DATA,__data
	.globl	_a                      ## @a
	.p2align	2
_a:
	.long	1                       ## 0x1
```
对比就可以发现, 带`static`修饰的`a`变量将不会有`.globl	_a`, 而使得`a`符号对外不可见.

内部静态变量是指写在方法体内静态变量, 它的赋值语句只会被执行一次, 比如:
```
#include <stdio.h>

void empty() {
    static int a = 1;
    a++;
}
```
其汇编实现:
```
	.section	__TEXT,__text,regular,pure_instructions
	.macosx_version_min 10, 12
	.globl	_empty
	.p2align	4, 0x90
_empty:                                 ## @empty
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
	movl	_empty.a(%rip), %eax
	addl	$1, %eax
	movl	%eax, _empty.a(%rip)
	popq	%rbp
	retq
	.cfi_endproc

	.section	__DATA,__data
	.p2align	2               ## @empty.a
_empty.a:
	.long	1                       ## 0x1


.subsections_via_symbols
```
代码中会生成一个仅对内可见的符号`_empty.a`, 而其实它也没有赋值的过程(`static int a = 1`没有赋值!), 而是在编译期就将初始值通过`.long	1`放入了数据段, 在代码中通过`movl	_empty.a(%rip), %eax`从数据段将值取出.

### 4.7 寄存器变量
是指强制要求变量被一直存放在寄存器中, 而不是像通常的局部变量一样放入栈中, 这样可以提高对此变量的访问性能. 注意, 关键字`register`可能会直接被编译器忽略! 有如下示例:
```
#include <stdio.h>

void empty()
{
    int a = 1;
	a++;
}

void empty_register()
{
    register int a = 1;
    a++;
}
```
其汇编实现如下:
```
	.section	__TEXT,__text,regular,pure_instructions
	.macosx_version_min 10, 12
	.globl	_empty
	.p2align	4, 0x90
_empty:                                 ## @empty
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
	movl	$1, -4(%rbp)
	movl	-4(%rbp), %eax
	addl	$1, %eax
	movl	%eax, -4(%rbp)
	popq	%rbp
	retq
	.cfi_endproc

	.globl	_empty_register
	.p2align	4, 0x90
_empty_register:                        ## @empty_register
	.cfi_startproc
## BB#0:
	pushq	%rbp
Ltmp3:
	.cfi_def_cfa_offset 16
Ltmp4:
	.cfi_offset %rbp, -16
	movq	%rsp, %rbp
Ltmp5:
	.cfi_def_cfa_register %rbp
	movl	$1, -4(%rbp)
	movl	-4(%rbp), %eax
	addl	$1, %eax
	movl	%eax, -4(%rbp)
	popq	%rbp
	retq
	.cfi_endproc


.subsections_via_symbols
```
会发现没啥区别, `register`指令被编译器无视了!  
_注：llvm不支持register变量原因是实现会很复杂且被认为没有太大用处。而gcc是支持的。_

### 4.8 块结构
快结构实际上将作用域做了进一步的细分, 而前面提到其实汇编没有变量作用域的概念, C语言不同块之间的变量在汇编中都存放在栈上的不同位置, 互不影响.

### 4.9 初始化
初始化的实例其实在前面的例子中都有所涉猎, 这里就做一个简单的总结吧.
* 局部变量的初始化, 通常是将整数放入寄存器然后放入栈中, 将浮点数从数据段放入寄存器然后放入栈中.
* 全局变量和静态变量的初始化, 通常是将数据放入预先编译到数据段, 使用时从数据段移动到寄存器然后进行运算.


### 4.10 递归
递归就是在方法内通过`call`指令调自己的符号.
第五章 指针和数组
===

在前面的文章中提到过, 在汇编层面, 所有的数据都存在栈/符号/寄存器中, 栈和符号都属于内存范畴, 进一步抽象就是, 所有的数据都在内存和寄存器中(这里先不讨论协处理器或者外部存储等其他范畴). 而指针的本质就是内存的地址!

### 5.1 指针和地址
指针通常被认为是c语言中比容易绕晕的一个特性, 而实际上指针的本质十分简单, 比如如下代码:
```
#include <stdio.h>

void empty()
{
   int a = 1;
   void *b = &a;
}
```
其汇编形式如下:
```
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
	leaq	-4(%rbp), %rax
	movl	$1, -4(%rbp)
	movq	%rax, -16(%rbp)
	popq	%rbp
	retq
	.cfi_endproc
```
从之前的介绍中知道变量`a`实际上是存放在栈上的, 在这里是`-4(%rbp)`, 其中rbp存的是栈底的地址, 那`-4(%rbp)`就是`栈底的地址 - 4`, 取地址操作`&a`实际上取到的就是`栈底的地址 - 4`这个值, `leaq	-4(%rbp), %rax`将值放入寄存器`rax`, `movq	%rax, -16(%rbp)`将rax存入变量`b`所在的`-16(%rbp)`里. 这样变量b里存的就是a的地址咯.  
也就是说, 如果一个变量b里存放的是另一个变量a的地址, 那么就可以称为, b是指向a的指针了!  
用汇编的说法来解释, 就是一个地址指向的内存里面存的内容是另一个地址!

在汇编层面有一个很重要的点就是要对思想做转变, 在汇编层面没有变量/对象和指针等概念, 只有寄存器/内存和数据, 而数据是没有类型的(数据只是一个比特序列)! 数据该怎么理解要看具体的汇编指令是怎么操作寄存器和内存的, 如果使用整数加法指令操作寄存器, 那么数据将被当做整数, 同理, 如果使用浮点数加法指令操作寄存器那么数据将被当做浮点数! 如果指令对寄存器做寻址操作, 那么数据将被当做地址(也就值指针)!

### 5.2 指针和函数参数
在第四章中已经介绍了函数的参数是通过寄存器和栈来传递的, 如果我们传递的参数是一个整数, 那么寄存器/栈中存的就是整数, 那如果我们传的参数是一个指针, 那么寄存器/栈中存的就是地址! 比如如下代码:
```
#include <stdio.h>

void swap(int *px, int *py) {
    int temp;
    temp = *px;
    *px = *py;
    *py = temp;
}

int main()
{
    int a = 1;
    int b = 2;
    swap(&a, &b);    
    return 0;
}
```
其汇编实现如下(节选):
```
# 调用部分
	leaq	-8(%rbp), %rdi   # &a, 取a地址
	leaq	-12(%rbp), %rsi  # &b, 取b地址
	movl	$0, -4(%rbp)
	movl	$1, -8(%rbp)     # 1放入a
	movl	$2, -12(%rbp)    # 2放入b
	callq	_swap

# 实现部分
	movq	%rdi, -8(%rbp)   # 参入放入栈, 作为变量px (地址)
	movq	%rsi, -16(%rbp)  # 参入放入栈, 作为变量py (地址)
	movq	-8(%rbp), %rsi   # px放入rsi
	movl	(%rsi), %eax     # rsi的值当做地址取数据放入eax (eax = *px)
	movl	%eax, -20(%rbp)  # eax放入栈, 作为变量temp
```
其关键流程就是: 取变量地址(栈, 提前分配的) -> 把地址作为参数调函数 -> 把参数作为地址存取数据.

### 5.3 指针和数组
数组是栈上提前分配好的一片内存, 如果你拿到了这片内存的地址(指针), 那么你将可以随意访问它.  
在日常的使用中, 可能经常会碰见如下两种访问数组的方法, 如`a[1]`和`*(a + 1)`, 它们有什么区别呢? 看代码:
```
#include <stdio.h>

void f1() {
    int a[2];
    int b = a[1];
}

void f2() {
    int a[2];
    int b = *(a+1);
}
```
其汇编实现:
```
_f1:                                    ## @f1
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
	subq	$32, %rsp
	movq	___stack_chk_guard@GOTPCREL(%rip), %rax
	movq	(%rax), %rax
	movq	%rax, -8(%rbp)
	movl	-12(%rbp), %ecx
	movl	%ecx, -20(%rbp)
	movq	___stack_chk_guard@GOTPCREL(%rip), %rax
	movq	(%rax), %rax
	movq	-8(%rbp), %rdx
	cmpq	%rdx, %rax
	jne	LBB0_2
## BB#1:
	addq	$32, %rsp
	popq	%rbp
	retq
LBB0_2:
	callq	___stack_chk_fail
	.cfi_endproc

	.globl	_f2
	.p2align	4, 0x90
_f2:                                    ## @f2
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
	movq	___stack_chk_guard@GOTPCREL(%rip), %rax
	movq	(%rax), %rax
	movq	%rax, -8(%rbp)
	movl	-12(%rbp), %ecx
	movl	%ecx, -20(%rbp)
	movq	___stack_chk_guard@GOTPCREL(%rip), %rax
	movq	(%rax), %rax
	movq	-8(%rbp), %rdx
	cmpq	%rdx, %rax
	jne	LBB1_2
## BB#1:
	addq	$32, %rsp
	popq	%rbp
	retq
LBB1_2:
	callq	___stack_chk_fail
	.cfi_endproc
```
完全一样有没有! 其中的关键点在于:
```
	movl	-12(%rbp), %ecx
	movl	%ecx, -20(%rbp)
```
这两句是`int b = a[1];`的实现, 二者均是数组头指针`-16(%rbp)`加上偏移量`4`的产物, 而这里`4`是`sizeof(int)`.

### 5.4 指针的算术运算
指针(地址)和整数很像, 可以做加减乘除等算术运算, 只不过在做运算的时候, 加减的增量的`1`是代表着一个元素的大小. 比如如下代码:
```
#include <stdio.h>

int main()
{
    int *a;
    a++;
    return 0;
}
```
其汇编实现(关键部分):
```
	movq	-16(%rbp), %rcx
	addq	$4, %rcx
	movq	%rcx, -16(%rbp)
```
即`a++`的实现, 加的数字是`4`而不是`1`, 因为`int`的大小是`4`.

在比如如下代码:
```
#include <stdio.h>

int main()
{
    char *a;
    a++;
    return 0;
}
```
其汇编实现(关键部分):
```
	movq	-16(%rbp), %rcx
	addq	$1, %rcx
	movq	%rcx, -16(%rbp)
```
即`a++`的实现, 加的数字是`1`, 因为`char`的大小是`1`.
第三章 控制流
===

在前面的章节中对控制流已经做了部分介绍, 本章会补足其他更全面的控制流的实现.

### 3.1 语句和块
在C中语句是用`;`作为分隔符的表达式, 而括号'{'和'}'将语句包裹成一个复合语句或者块. 那块的底层实现是什么呢? 为了方便对比, 提取上一章中的一个例子做一个简单的修改:
```
#include <stdio.h>

main()
{
    int a,b,c;
		{
        a = a | b & c;
		}
}
```
其汇关键编代码如下:
```
	movl	-4(%rbp), %ecx
	movl	-8(%rbp), %edx
	andl	-12(%rbp), %edx
	orl	%edx, %ecx
	movl	%ecx, -4(%rbp)
```
将汇编代码和没有加括号的上一章的例子做对比, 会发现没有任何区别, 也就意味着'{'和'}'可以被当做C语言用来组织代码或者规划作用域的语法糖.

### 3.2 if-else
if-else的基本实现在前面章节已经做过介绍, 这里讲会对更多细节做介绍, 比如经常使用的:
```
#include <stdio.h>

main()
{
    int a = 1;
		if (a != 0) {}
}
```
经常会被简写为:
```
#include <stdio.h>

main()
{
    int a = 1;
		if (a) {}
}
```
它们的汇编实现会有区别么? 先看第一种实现的关键部分:
```
	movl	$1, -8(%rbp)
	cmpl	$0, -8(%rbp)
	je	LBB0_2
## BB#1:
	jmp	LBB0_2
LBB0_2:
	movl	-4(%rbp), %eax
```
其实现是将`1`放入变量`a`的栈空间, 然后将`0`和`a`的值做比较, 判断是否相等然后走到不同的分支.  
再看第二种简写实现的关键部分:
```
	movl	$1, -8(%rbp)
	cmpl	$0, -8(%rbp)
	je	LBB0_2
## BB#1:
	jmp	LBB0_2
LBB0_2:
	movl	-4(%rbp), %eax
```
毫无差别! `if (a) {}`也是C语言给`if (a != 0) {}`的语法糖!

前面的例子都只有`if`没有`else`, 再看看带`else`的实现吧:
```
#include <stdio.h>

main()
{
    int a = 1;
		if (a) {
		    a = 2;
		} else {
		    a = 3;
		}
}
```
其关键汇编代码为:
```
	movl	$1, -8(%rbp)
	cmpl	$0, -8(%rbp)
	je	LBB0_2
## BB#1:
	movl	$2, -8(%rbp)
	jmp	LBB0_3
LBB0_2:
	movl	$3, -8(%rbp)
LBB0_3:
	movl	-4(%rbp), %eax
```
将`1`放入变量`a`的栈空间, 然后与`0`做对比, `je LBB0_2`会在数值相等(`a == 0`)的情况下跳转到`LBB0_2`处, 将`3`放入变量`a`, 顺序执行到`LBB0_3`结尾处, 这里就是else的情况. 如果不相等(`a != 0`)则会继续往下执行, 将`2`放入变量`a`, 然后无条件跳转到结尾处`LBB0_3`.

### 3.3 else if
看完`if`和`else`, 再来看看`else if`的实现:
```
#include <stdio.h>

main()
{
    int a = 1;
		if (a == 1) {
		    a = 11;
		} else if (a == 2) {
		    a = 12;
		} else {
		    a = 13;
		}
}
```
其关键汇编代码如下:
```
	movl	$1, -8(%rbp)
	cmpl	$1, -8(%rbp)
	jne	LBB0_2
## BB#1:
	movl	$11, -8(%rbp)
	jmp	LBB0_6
LBB0_2:
	cmpl	$2, -8(%rbp)
	jne	LBB0_4
## BB#3:
	movl	$12, -8(%rbp)
	jmp	LBB0_5
LBB0_4:
	movl	$13, -8(%rbp)
LBB0_5:
	jmp	LBB0_6
LBB0_6:
	movl	-4(%rbp), %eax
```
将`1`放入变量`a`的栈空间, 将`1`与`a`做比较, 如果相等则顺序执行下一句, 将`11`放入`a`后跳转到结尾`LBB0_6`完成`a = 11;`. 如果不相等则跳转到`LBB0_2`, 继续将`2`与`a`做对比, 如果相等则顺序执行下一句将`12`放入`a`后跳转到结尾`LBB0_6`完成`a = 12`. 如果不相等则跳转到`LBB0_4`, 将`13`放入`a`完成`a = 13;`后跳转到`LBB0_6`处.  
同学们应该会发现`LBB0_5`处的`jmp	LBB0_6`完全没啥用途, 就算不跳转顺序执行的也是到`LBB0_6`处!

### 3.4 switch
`switch`的实现与`if else`的实现有挺大的不同, 有如下代码:
```
#include <stdio.h>

main()
{
    int a;
		switch (a) {
		    case 1:
		        a = 11;
						break;
		
		    case 2:
		        a = 12;
						break;
				
				default:
		        a = 13;
		}
}
```
其关键汇编代码如下:
```
	movl	-8(%rbp), %eax
	movl	%eax, %ecx
	subl	$1, %ecx
	movl	%eax, -12(%rbp)         ## 4-byte Spill
	movl	%ecx, -16(%rbp)         ## 4-byte Spill
	je	LBB0_1
	jmp	LBB0_5
LBB0_5:
	movl	-12(%rbp), %eax         ## 4-byte Reload
	subl	$2, %eax
	movl	%eax, -20(%rbp)         ## 4-byte Spill
	je	LBB0_2
	jmp	LBB0_3
LBB0_1:
	movl	$11, -8(%rbp)
	jmp	LBB0_4
LBB0_2:
	movl	$12, -8(%rbp)
	jmp	LBB0_4
LBB0_3:
	movl	$13, -8(%rbp)
LBB0_4:
	movl	-4(%rbp), %eax
```
将变量`a`的值取出放入`eax`(这里`a`未初始化, 目的在于避免编译器将`switch`做优化), 在把`eax`放入`ecx`, 将`ecx`减去`1`(也就是`a - 1`), `sub`指令会在减法的同时设置`flag`(NZCV)用于做分支判断, 这里如果`a = 1` 那么`a - 1 == 0`则flag`Z = 0`也就是条件代码`e`. 将`eax`和`ecx`暂存到栈里, 指令`je LBB0_1`如果条件`e`满足(`a == 1`)则跳转到`LBB0_1`, 完成`a = 11`语句后跳转到结尾`LBB0_4`; 如果条件`e`不满足(`a != 1`)则继续往下执行`LBB0_5`. 将之前保存的`eax`也就是`a`从栈中取出, 与`2`做减法, 判断是否等于`2`, 如果等于则跳转到`LBB0_2`完成`a = 12`语句后跳到结尾; 如果不等于`2`则继续向下执行`jmp	LBB0_3`跳转到`LBB0_3`完成`a = 13`语句, 然顺序执行到结尾.

可以看出`switch`的实现和`if else`实现的差别在于, `switch`是一次判断跟着两个跳转, 条件满足跳分支1不满足跳分支2; 而`if else`是一次判断跟着一次跳转, 条件满足调分支1不满足则继续执行下一次判断.

### 3.5 while和for循环
先来个`while`循环的例子:
```
#include <stdio.h>

main()
{
    int a = 0;
    while (a < 10) {
        a++;
    }
}
```
其关键汇编代码为:
```
	movl	$0, -8(%rbp)
LBB0_1:                                 ## =>This Inner Loop Header: Depth=1
	cmpl	$10, -8(%rbp)
	jge	LBB0_3
## BB#2:                                ##   in Loop: Header=BB0_1 Depth=1
	movl	-8(%rbp), %eax
	addl	$1, %eax
	movl	%eax, -8(%rbp)
	jmp	LBB0_1
LBB0_3:
	movl	-4(%rbp), %eax
```
将`0`赋给变量`a`, 将`a`和`10`做比较, 如果大于等于则跳到结束`LBB0_3`, 如果小于等于则顺序执行到`a++`, 再跳回`LBB0_1`继续做比较.

在第一章1.3节中已经对`for`的实现做了简介, 这里再介绍一下for的另一种情况, 无限循环:
```
#include <stdio.h>

main()
{
    int a = 0;
    for (;;) {
        a++;
    }
}
```
其关键汇编代码为:
```
LBB0_1:                                 ## =>This Inner Loop Header: Depth=1
	movl	-8(%rbp), %eax
	addl	$1, %eax
	movl	%eax, -8(%rbp)
	jmp	LBB0_1
```
将变量`a`移动到`eax`, 对`eax`加`1`再放回变量`a`, 完成`a++;`语句, 再跳回`LBB0_1`处. 从代码中可以看出, 这里循环没有跳出到其他代码的机会, 会永远执行下去!

### 3.6 do-while循环
`do-while`循环实际上是while循环的变种写法, 其实现机制很类似. 有如下示例代码:
```
#include <stdio.h>

main()
{
    int a = 0;
    do {
        a++;
    }
    while (a < 10);
}
```
其关键汇编代码如下:
```
	movl	$0, -8(%rbp)
LBB0_1:                                 ## =>This Inner Loop Header: Depth=1
	movl	-8(%rbp), %eax
	addl	$1, %eax
	movl	%eax, -8(%rbp)
## BB#2:                                ##   in Loop: Header=BB0_1 Depth=1
	cmpl	$10, -8(%rbp)
	jl	LBB0_1
## BB#3:
	movl	-4(%rbp), %eax
```
将`0`赋给变量`a`, 完成`a++`, 比较`a`和`10`, 如果小于则跳到`LBB0_1`继续`a++`, 否则顺序执行到循环外. 可以看出`while`和`do-while`的差别在于, 一个判断在前运算灾后, 一个运算在判断在后.

### 3.7 break和continue
经过了前面对循环的介绍, `break`和`continue`的实现原理大家应该可以猜到, `break`是直接跳转到循环外, 而`continue`是跳转到进行比较的地方.  
先看看`break`:
```
#include <stdio.h>

main()
{
    int a = 0;
    for (;;) {
        a++;
        if (a==10) {break;}
    }
}
```
其关键汇编代码为:
```
	movl	$0, -8(%rbp)
LBB0_1:                                 ## =>This Inner Loop Header: Depth=1
	movl	-8(%rbp), %eax
	addl	$1, %eax
	movl	%eax, -8(%rbp)
	cmpl	$10, -8(%rbp)
	jne	LBB0_3
## BB#2:
	jmp	LBB0_4
LBB0_3:                                 ##   in Loop: Header=BB0_1 Depth=1
	jmp	LBB0_1
LBB0_4:
	movl	-4(%rbp), %eax
```
将`0`放入变量`a`, 运算`a++`, 比较`a`和`10`, 如果不等于则跳到`LBB0_3`, 然后跳到`LBB0_1`处继续`a++`, 如果等于则顺序执行到`## BB#2`处, 跳转到循环结束`LBB0_4`. `break`就是通过`jmp`指令跳转到循环外.

再看看`continue`:
```
#include <stdio.h>

main()
{
    int a = 0;
    for (int i = 0; i<100; i++) {
        if (a == 10) {continue;}
        a++;
    }
}
```
其关键汇编代码为:
```
	movl	$0, -8(%rbp)
	movl	$0, -12(%rbp)
LBB0_1:                                 ## =>This Inner Loop Header: Depth=1
	cmpl	$100, -12(%rbp)
	jge	LBB0_6
## BB#2:                                ##   in Loop: Header=BB0_1 Depth=1
	cmpl	$10, -8(%rbp)
	jne	LBB0_4
## BB#3:                                ##   in Loop: Header=BB0_1 Depth=1
	jmp	LBB0_5
LBB0_4:                                 ##   in Loop: Header=BB0_1 Depth=1
	movl	-8(%rbp), %eax
	addl	$1, %eax
	movl	%eax, -8(%rbp)
LBB0_5:                                 ##   in Loop: Header=BB0_1 Depth=1
	movl	-12(%rbp), %eax
	addl	$1, %eax
	movl	%eax, -12(%rbp)
	jmp	LBB0_1
LBB0_6:
	movl	-4(%rbp), %eax
```
将`0`赋给变量`a`和`i`, 比较`i`和`100`, 如果大于等于则跳到循环结尾`LBB0_6`处, 否则顺序执行到`## BB#2`处比较`a`和`10`, 如果不等于则跳到`LBB0_4`处运算`a++;`后运算`i++;`然后回到循环头部, 如果等于则顺序执行到`## BB#3`跳转到`LBB0_5`运算`i++`后回到循环头部.

### 3.8 goto和label
在第一章中对于`label`就已经做了介绍, 而对于`goto`也可以很容易的联想到'jmp'指令, 那C语言的`label`和`goto`与汇编的`label`和`jmp`一样么? 看代码:
```
#include <stdio.h>

main()
{
    int a = 0;
    for (;;) {
        a++;
        if (a == 10) {goto end;} 
    }
    a += 2;
end:
    a += 3;
}
```
其关键汇编代码为:
```
	movl	$0, -8(%rbp)
LBB0_1:                                 ## =>This Inner Loop Header: Depth=1
	movl	-8(%rbp), %eax
	addl	$1, %eax
	movl	%eax, -8(%rbp)
	cmpl	$10, -8(%rbp)
	jne	LBB0_3
## BB#2:
	jmp	LBB0_4
LBB0_3:                                 ##   in Loop: Header=BB0_1 Depth=1
	jmp	LBB0_1
LBB0_4:
	movl	-8(%rbp), %eax
	addl	$3, %eax
	movl	%eax, -8(%rbp)
```
将`0`赋给变量`a`, 进入循环计算`a++`, 然后比较`a`和`10`, 如果不相等则跳转到`LBB0_3`后跳转回`LBB0_1`继续循环, 如果相等则顺序执行`## BB#2`跳转到`LBB0_4`进行`a += 3`运算. 值得关注的是`goto end;`代码中的`end`标签被编译成了`LBB0_4`, 而永远走不到的`a += 2;`语句被编译器优化掉了.
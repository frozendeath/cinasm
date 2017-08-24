第七章 人肉汇编器和反汇编器
===

挑战极限吧少年！
### 7.1 人肉汇编器
先来个简单的例子，大家可以尝试翻译一下大概的汇编：
```
#include <stdio.h>

main() {
    int a, b;
	a = 1;
	b = a + 1;
}
```
其汇编大概是这样的：
* 给a，b在栈上隐式地预留2个int大小的空间
* 把1放入a的栈空间
* 把a的栈空间的数据放到寄存器里面
* 对寄存器做加1
* 把结果放入b的栈空间


在来个复杂点的例子， 大家再试试：
```
#include <stdio.h>

main() {
    int c;
    while ((c = getchar()) != EOF) {
        putchar(tolower(c));
	}
}
```
其汇编大概是这样的：
* 给c分配一个栈上的空间
* 立一个label在接下来的代码的头部，方便跳回来
* call getchar方法，返回值rax放入c的栈空间
* 把c的栈空间的内容捞出来跟EOF做比较
* 如果不等于则往下走，如果等于则跳到循环尾部
* 往下走就是call tolower参数是c（会放到rax，返回值也在rax），再call putchar（参数是rax），跳回最开始立的label
* 结束


### 7.2 人肉反汇编器
其实主要考猜和工具辅助，工具辅助会帮助给出更多的符号或者字面量意思，让汇编代码更容易理解。随便拉了个库看看汇编：
```
             _libusb_get_bus_number:
0000130c         push       ebp
0000130d         mov        ebp, esp
0000130f         mov        eax, dword [ss:ebp+arg_0]
00001312         movzx      eax, byte [ds:eax+0x34]
00001316         leave      
00001317         ret 
```
方法名_libusb_get_bus_number，取bus number, 代码里面前两行prolog无视，最后的leave是epilog无视，中间的mov指令是把传进来的参数从栈里面取出来放入eax，下面的movzx是把eax+0x34的数据取出来放入eax，相当于 `eax = (int)*(int8_t *)(eax+0x34)`, 返回eax。一开始没把eax入栈说明参数很可能是个很大的数据结构直接就用栈来传了。

再来个复杂点的：
```
             _usbi_get_device_by_session_id:
00001770         push       ebp                                                 ; XREF=_darwin_get_device_list+178
00001771         mov        ebp, esp
00001773         push       edi
00001774         push       esi
00001775         push       ebx
00001776         sub        esp, 0x1c
00001779         mov        esi, dword [ss:ebp+arg_0]
0000177c         mov        ebx, dword [ss:ebp+arg_4]
0000177f         lea        edi, dword [ds:esi+0x18]
00001782         mov        dword [ss:esp+0x28+var_28], edi                     ; argument "mutex" for method imp___symbol_stub__pthread_mutex_lock
00001785         call       imp___symbol_stub__pthread_mutex_lock
0000178a         mov        edx, dword [ds:esi+0x14]
0000178d         sub        edx, 0x38
00001790         lea        ecx, dword [ds:esi+0x10]
00001793         jmp        0x17a4

00001795         cmp        dword [ds:edx+0x40], ebx                            ; XREF=_usbi_get_device_by_session_id+57
00001798         jne        0x179e

0000179a         mov        ebx, edx
0000179c         jmp        0x17ad

0000179e         mov        edx, dword [ds:edx+0x3c]                            ; XREF=_usbi_get_device_by_session_id+40
000017a1         sub        edx, 0x38

000017a4         lea        eax, dword [ds:edx+0x38]                            ; XREF=_usbi_get_device_by_session_id+35
000017a7         cmp        eax, ecx
000017a9         jne        0x1795

000017ab         xor        ebx, ebx

000017ad         mov        dword [ss:esp+0x28+var_28], edi                     ; argument "mutex" for method imp___symbol_stub__pthread_mutex_unlock, XREF=_usbi_get_device_by_session_id+44
000017b0         call       imp___symbol_stub__pthread_mutex_unlock
000017b5         mov        eax, ebx
000017b7         add        esp, 0x1c
000017ba         pop        ebx
000017bb         pop        esi
000017bc         pop        edi
000017bd         leave      
000017be         ret    
```
无视prolog。
```
00001773         push       edi
00001774         push       esi
00001775         push       ebx
```
参数入栈。
```
00001779         mov        esi, dword [ss:ebp+arg_0]
0000177c         mov        ebx, dword [ss:ebp+arg_4]
0000177f         lea        edi, dword [ds:esi+0x18]
00001782         mov        dword [ss:esp+0x28+var_28], edi                     ; argument "mutex" for method imp___symbol_stub__pthread_mutex_lock
00001785         call       imp___symbol_stub__pthread_mutex_lock
```
取出mutex_lock，然后锁一把。
```
0000178a         mov        edx, dword [ds:esi+0x14]
0000178d         sub        edx, 0x38
00001790         lea        ecx, dword [ds:esi+0x10]
```
从esi也就是arg_0中偏移量0x14的地方取了个东西出来（东西1），这个东西减去了0x38。然后又从arg0里面偏移量0x10取了个东西出来（东西2）。
```
0000178a         mov        edx, dword [ds:esi+0x14]
0000178d         sub        edx, 0x38
00001790         lea        ecx, dword [ds:esi+0x10]
00001793         jmp        0x17a4

00001795         cmp        dword [ds:edx+0x40], ebx                            ; XREF=_usbi_get_device_by_session_id+57
00001798         jne        0x179e

0000179a         mov        ebx, edx
0000179c         jmp        0x17ad

0000179e         mov        edx, dword [ds:edx+0x3c]                            ; XREF=_usbi_get_device_by_session_id+40
000017a1         sub        edx, 0x38

000017a4         lea        eax, dword [ds:edx+0x38]                            ; XREF=_usbi_get_device_by_session_id+35
000017a7         cmp        eax, ecx
000017a9         jne        0x1795

000017ab         xor        ebx, ebx

000017ad         mov        dword [ss:esp+0x28+var_28], edi                     ; argument "mutex" for method imp___symbol_stub__pthread_mutex_unlock, XREF=_usbi_get_device_by_session_id+44
```
然后直接就跳走了，然后进行比较，这里很像在做while循环。  
这里把东西1+0x38偏移量里的内容取出来，跟东西2做对比。  
如果相等，则ebx设为0并unlock，这里的ebx在后面的代码就是返回值。这一步对比看起来像是一个默认值对比。  
如果不相等则继续把参数2（arg_4）跟东西1（edx）偏移0x40的内容做对比，如果相等则edx放入ebx，并跳到解锁，这里的ebx仍然是返回值，从方法名看，edx里面存的应该就是device结构体了，而device结构体的偏移0x40应该session_id。如果不相等则跳到`0000179e mov edx, dword [ds:edx+0x3c]`这里，这里相当于edx也就是device的偏移-0x38+0x3c地方里面存的也是一个device，也就意味着device可能自己做了一个list，来维护所有的device信息。  
这里比较纠结的地方在于，edx先减去0x38，又再取值的地方加上0x38来取，很可能意味着device的偏移0x38位置是session_id，因为它用的比较多，编译器做了优化。

从网上搜了定义:
```
struct list_head {
struct list_head *prev, *next;
};


struct libusb_device {

pthread_mutex_t lock;
int refcnt;
 
struct libusb_context *ctx;
 
uint8_t bus_number;
uint8_t device_address;
uint8_t num_configurations;
 
struct list_head list;
unsigned long session_data;
unsigned char os_priv[0];
 
};
```
看起来很像，0x38处就是list（list中的prev），0x3c就是list中的next, 0x40就是session_id。

而东西2来自arg_1：
```
struct libusb_device *usbi_get_device_by_session_id(struct libusb_context *ctx,
unsigned long session_id);

struct libusb_context {
 
int debug;
int debug_fixed;
 
/* internal control pipe, used for interrupting event handling when
 * something needs to modify poll fds. */
int ctrl_pipe[2];
 
struct list_head usb_devs;
pthread_mutex_t usb_devs_lock;
......
}
```

arg_1是libusb_context，而context的0x10处是usb_devs的prev，假如usb_devs是list，那么prev应该是NULL或者一个特定的空标识，那第一次比较就是比较的空或者默认标识，而返回了0(也就是NULL)。
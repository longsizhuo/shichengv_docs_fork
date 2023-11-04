# 链接器的动态链接过程

本篇是对于操作系统如何加载动态库过程的分析，通过在真实的机器上运行的结果并结合链接器源代码来一步一步的分析动态链接的这个过程。对于Windows系统的PE格式文件和`dll`库文件，它们的机制linux也是类似的。不过Windows没有源代码，只能结合汇编分析，所以最终还是选择在Linux上讲解。

主要讲解动态链接这个过程，并着重分析`_dl_fixup()`函数的前半部分。(因为水平有限)

- glibc库版本：2.37 (Arch Linux的 2.38版本没有调试符号表，不能直接在gdb里面打印变量的信息，分析起来比较麻烦。)

*** 

## 源代码

```c
// filename: main2.c

#include <stdio.h>
#include "vector.h"

int x[2] = {1, 2};
int y[2] = {3, 4};
int z[2];

int main(){
    char hello[] = "hello, world";
    addvec(x, y, z, 2);
    printf("%s\n", hello);
    printf("z = [%d %d]\n", z[0],z[1]);
    puts("");

}
```

```c
// filename: addvec.c

int addcnt = 0;

void addvec(int *x, int *y, int *z, int n){
    int i;
    addcnt++;
    for (i = 0; i < n; i++) 
        z[i] = x[i] + y[i];
    
}
```

```c
// filename: multvec.c

int multcnt = 0;
void multvec(int *x, int *y, int *z, int n){
    int i;
    multcnt++;
    for (i = 0; i < n; i++)
        z[i] = x[i] * y[i];
}
```

```c
// filename: vector.h

void addvec(int *x, int *y, int *z, int n);
void multvec(int *x, int *y, int *z, int n);
```

**编译选项**
```bash
gcc -shared -fpic -o libvector.so src/addvec.c src/multvec.c
gcc -Og -z lazy -fno-stack-protector -o prog2l src/main2.c ./libvector.so
```

- `-z lazy` 开启延迟绑定
- `-fno-stack-protector` 禁用栈保护，这样.got.plt表中就没有栈保护函数。
- `-fpic` 开启PIC可重定位目标函数调用

## lazy binding

## start
在终端上输入以下命令，让我们开始吧。
```bash
$ gdb -q prog2l
Reading symbols from prog2l...
(No debugging symbols found in prog2l)
(gdb) start
Temporary breakpoint 1 at 0x1159
Starting program: /media/shicheng/shichengv/sources/lazybinding/prog2l 
[Thread debugging using libthread_db enabled]
Using host libthread_db library "/lib/x86_64-linux-gnu/libthread_db.so.1".

Temporary breakpoint 1, 0x0000555555555159 in main ()
```
```
Temporary breakpoint 1 at 0x1159
Temporary breakpoint 1, 0x0000555555555159 in main ()
```

根据上面这两条信息就可以得出，该程序程序被加载到虚拟内存中的起始页位置是 0x0000555555554000 

使用下面这条命令可以显示当前二进制程序的一些信息。这些信息在后面将会用到。
```bash
(gdb) info files
Symbols from "/home/kali/lazybinding/prog2l".                                                                                                                                                                                               
Native process:                                                                                                                                                                                                                             
        Using the running image of child Thread 0x7ffff7dc3740 (LWP 211202).                                                                                                                                                                
        While running this, GDB does not access memory from...                                                                                                                                                                              
Local exec file:                                                                                                                                                                                                                            
        `/home/kali/lazybinding/prog2l', file type elf64-x86-64.                                                                                                                                                                            
        Entry point: 0x555555555060                                                                                                                                                                                                         
        0x0000555555554318 - 0x0000555555554334 is .interp                                                                                                                                                                                  
        0x0000555555554338 - 0x0000555555554378 is .note.gnu.property                                                                                                                                                                       
        0x0000555555554378 - 0x000055555555439c is .note.gnu.build-id                                                                                                                                                                       
        0x000055555555439c - 0x00005555555543bc is .note.ABI-tag                                                                                                                                                                            
        0x00005555555543c0 - 0x00005555555543dc is .gnu.hash                                                                                                                                                                                
        0x00005555555543e0 - 0x00005555555544b8 is .dynsym                                                                                                                                                                                  
        0x00005555555544b8 - 0x0000555555554562 is .dynstr                                                                                                                                                                                  
        0x0000555555554562 - 0x0000555555554574 is .gnu.version                                                                                                                                                                             
        0x0000555555554578 - 0x00005555555545a8 is .gnu.version_r
        0x00005555555545a8 - 0x0000555555554668 is .rela.dyn
        0x0000555555554668 - 0x00005555555546b0 is .rela.plt
        0x0000555555555000 - 0x000055555555501b is .init
        0x0000555555555020 - 0x0000555555555060 is .plt
        0x0000555555555060 - 0x00005555555551d7 is .text
        0x00005555555551d8 - 0x00005555555551e5 is .fini
        0x0000555555556000 - 0x0000555555556011 is .rodata
        0x0000555555556014 - 0x0000555555556038 is .eh_frame_hdr
        0x0000555555556038 - 0x00005555555560ac is .eh_frame
        0x0000555555557dc0 - 0x0000555555557dc8 is .init_array
        0x0000555555557dc8 - 0x0000555555557dd0 is .fini_array
        0x0000555555557dd0 - 0x0000555555557fc0 is .dynamic
        0x0000555555557fc0 - 0x0000555555557fe8 is .got
        0x0000555555557fe8 - 0x0000555555558018 is .got.plt
        0x0000555555558018 - 0x0000555555558038 is .data
        0x0000555555558038 - 0x0000555555558048 is .bss

        ......

        0x00007ffff7fcb238 - 0x00007ffff7fcb25c is .note.gnu.build-id in /lib64/ld-linux-x86-64.so.2
        0x00007ffff7fcb260 - 0x00007ffff7fcb398 is .hash in /lib64/ld-linux-x86-64.so.2
        0x00007ffff7fcb398 - 0x00007ffff7fcb4f4 is .gnu.hash in /lib64/ld-linux-x86-64.so.2
        0x00007ffff7fcb4f8 - 0x00007ffff7fcb8a0 is .dynsym in /lib64/ld-linux-x86-64.so.2
        0x00007ffff7fcb8a0 - 0x00007ffff7fcbb51 is .dynstr in /lib64/ld-linux-x86-64.so.2
        0x00007ffff7fcbb52 - 0x00007ffff7fcbba0 is .gnu.version in /lib64/ld-linux-x86-64.so.2
        0x00007ffff7fcbba0 - 0x00007ffff7fcbc8c is .gnu.version_d in /lib64/ld-linux-x86-64.so.2
        0x00007ffff7fcbc90 - 0x00007ffff7fcbcd8 is .rela.dyn in /lib64/ld-linux-x86-64.so.2
        0x00007ffff7fcbcd8 - 0x00007ffff7fcbcf0 is .relr.dyn in /lib64/ld-linux-x86-64.so.2
        0x00007ffff7fcc000 - 0x00007ffff7ff07b1 is .text in /lib64/ld-linux-x86-64.so.2
        0x00007ffff7ff1000 - 0x00007ffff7ff6eb8 is .rodata in /lib64/ld-linux-x86-64.so.2
        0x00007ffff7ff6eb8 - 0x00007ffff7ff77cc is .eh_frame_hdr in /lib64/ld-linux-x86-64.so.2
        0x00007ffff7ff77d0 - 0x00007ffff7ffab70 is .eh_frame in /lib64/ld-linux-x86-64.so.2
        0x00007ffff7ffb940 - 0x00007ffff7ffce60 is .data.rel.ro in /lib64/ld-linux-x86-64.so.2
        0x00007ffff7ffce60 - 0x00007ffff7ffcfc0 is .dynamic in /lib64/ld-linux-x86-64.so.2
        0x00007ffff7ffcfc0 - 0x00007ffff7ffcfd0 is .got in /lib64/ld-linux-x86-64.so.2
        0x00007ffff7ffcfe8 - 0x00007ffff7ffd000 is .got.plt in /lib64/ld-linux-x86-64.so.2
        0x00007ffff7ffd000 - 0x00007ffff7ffe0e8 is .data in /lib64/ld-linux-x86-64.so.2
        0x00007ffff7ffe0f0 - 0x00007ffff7ffe2c8 is .bss in /lib64/ld-linux-x86-64.so.2  

        ......

        0x00007ffff7fbe2a8 - 0x00007ffff7fbe2d8 is .note.gnu.property in ./libvector.so                                                                                                                                                     
        0x00007ffff7fbe2d8 - 0x00007ffff7fbe2fc is .note.gnu.build-id in ./libvector.so
        0x00007ffff7fbe300 - 0x00007ffff7fbe334 is .gnu.hash in ./libvector.so
        0x00007ffff7fbe338 - 0x00007ffff7fbe410 is .dynsym in ./libvector.so
        0x00007ffff7fbe410 - 0x00007ffff7fbe499 is .dynstr in ./libvector.so
        0x00007ffff7fbe49a - 0x00007ffff7fbe4ac is .gnu.version in ./libvector.so
        0x00007ffff7fbe4b0 - 0x00007ffff7fbe4d0 is .gnu.version_r in ./libvector.so
        0x00007ffff7fbe4d0 - 0x00007ffff7fbe5a8 is .rela.dyn in ./libvector.so
        0x00007ffff7fbf000 - 0x00007ffff7fbf01b is .init in ./libvector.so
        0x00007ffff7fbf020 - 0x00007ffff7fbf1f5 is .text in ./libvector.so
        0x00007ffff7fbf1f8 - 0x00007ffff7fbf205 is .fini in ./libvector.so
        0x00007ffff7fc0000 - 0x00007ffff7fc001c is .eh_frame_hdr in ./libvector.so
        0x00007ffff7fc0020 - 0x00007ffff7fc007c is .eh_frame in ./libvector.so
        0x00007ffff7fc1e28 - 0x00007ffff7fc1e30 is .init_array in ./libvector.so
        0x00007ffff7fc1e30 - 0x00007ffff7fc1e38 is .fini_array in ./libvector.so
        0x00007ffff7fc1e38 - 0x00007ffff7fc1fb8 is .dynamic in ./libvector.so
        0x00007ffff7fc1fb8 - 0x00007ffff7fc1fe8 is .got in ./libvector.so
        0x00007ffff7fc1fe8 - 0x00007ffff7fc2000 is .got.plt in ./libvector.so
        0x00007ffff7fc2000 - 0x00007ffff7fc2008 is .data in ./libvector.so
        0x00007ffff7fc2008 - 0x00007ffff7fc2018 is .bss in ./libvector.so

        ......


```

上面展示了主要介绍的几个程序：`prog2l`, `libvector.so`, `ld-linux-x86-64.so.2`。
> `.got` 是可重定位的全局变量，`.got.plt`与`.plt`配合使用，重定位目标函数。

[https://stackoverflow.com/questions/11676472/what-is-the-difference-between-got-and-got-plt-section]

我们先把注意力放在PIC函数调用的过程上。

***
用gdb查看一下main函数的反汇编。

```
(gdb) disass main
Dump of assembler code for function main:
=> 0x0000555555555159 <+0>:     sub    $0x18,%rsp
   0x000055555555515d <+4>:     movabs $0x77202c6f6c6c6568,%rax
   0x0000555555555167 <+14>:    mov    %rax,0x3(%rsp)
   0x000055555555516c <+19>:    movabs $0x646c726f77202c,%rax
   0x0000555555555176 <+29>:    mov    %rax,0x8(%rsp)
   0x000055555555517b <+34>:    mov    $0x2,%ecx
   0x0000555555555180 <+39>:    lea    0x2eb9(%rip),%rdx        # 0x555555558040 <z>
   0x0000555555555187 <+46>:    lea    0x2e9a(%rip),%rsi        # 0x555555558028 <y>
   0x000055555555518e <+53>:    lea    0x2e9b(%rip),%rdi        # 0x555555558030 <x>
   0x0000555555555195 <+60>:    call   0x555555555050 <addvec@plt>
   0x000055555555519a <+65>:    lea    0x3(%rsp),%rdi
   0x000055555555519f <+70>:    call   0x555555555030 <puts@plt>
   0x00005555555551a4 <+75>:    mov    0x2e9a(%rip),%edx        # 0x555555558044 <z+4>
   0x00005555555551aa <+81>:    mov    0x2e90(%rip),%esi        # 0x555555558040 <z>
   0x00005555555551b0 <+87>:    lea    0xe4d(%rip),%rdi        # 0x555555556004
   0x00005555555551b7 <+94>:    mov    $0x0,%eax
   0x00005555555551bc <+99>:    call   0x555555555040 <printf@plt>
   0x00005555555551c1 <+104>:   lea    0xe48(%rip),%rdi        # 0x555555556010
   0x00005555555551c8 <+111>:   call   0x555555555030 <puts@plt>
   0x00005555555551cd <+116>:   mov    $0x0,%eax
   0x00005555555551d2 <+121>:   add    $0x18,%rsp
   0x00005555555551d6 <+125>:   ret
End of assembler dump.
```

由于使用了动态链接库，所以此时调用call addvec并不是真正的addvec函数地址，从地址上就可以看出来(call   0x555555555050 <addvec@plt>)。在`0x555555555195`打上断点，运行程序，然后单步跳入到函数内部，再次观察此时的函数内容。

```
(gdb) b *  0x0000555555555195
Breakpoint 2 at 0x555555555195
(gdb) continue 
Continuing.

Breakpoint 2, 0x0000555555555195 in main ()

(gdb) si
0x0000555555555050 in addvec@plt ()

(gdb) disass
Dump of assembler code for function addvec@plt:
=> 0x0000555555555050 <+0>:     jmp    *0x2fba(%rip)        # 0x555555558010 <addvec@got.plt>
   0x0000555555555056 <+6>:     push   $0x2
   0x000055555555505b <+11>:    jmp    0x555555555020
End of assembler dump.
```

此时`addvec`函数并不是真正的addvec函数地址，从程序的执行流可以看出来，`jmp`只是单纯的跳到了下一条汇编指令。
```
jmp    *0x2fba(%rip)
```
这句代码跳转到 rip + 0x2fba 的位置存储的地址中。实际上 rip + 0x2fba 的值就等于 `0x0000555555558010` 也就是 .got.plt[5]的地址，这个地址存储的值就是 `0x0000555555555056`, 也就是当前程序执行流的下一条指令。

> 关于 rip 的值的补充，rip存储了下一条指令的地址。此时程序执行流的下一条指令是 `jmp    *0x2fba(%rip)` 所以，此时rip的值是 `0x555555555050`。当程序执行`jmp    *0x2fba(%rip)` 时，rip的值应该是程序执行流的下一条指令，也就是 `0x555555555056`，将该值 + 0x2FBA 的值 就是 0x555555558010 。

```
(gdb) disass
Dump of assembler code for function addvec@plt:
=> 0x0000555555555050 <+0>:     jmp    *0x2fba(%rip)        # 0x555555558010 <addvec@got.plt>
   0x0000555555555056 <+6>:     push   $0x2
   0x000055555555505b <+11>:    jmp    0x555555555020
End of assembler dump.
(gdb) info reg rip
rip            0x555555555050      0x555555555050 <addvec@plt>
(gdb) print /x (0x555555555056+0x2fba)
$6 = 0x555555558010
(gdb) x /gx 0x555555558010
0x555555558010 <addvec@got.plt>:        0x0000555555555056
```

上面我们已经观察到了`.got.plt`节的信息。
``` 
0x0000555555557fe8 - 0x0000555555558018 is .got.plt
```
下面展示了.got.plt节的存储的所有信息。.got.plt[5]对应的就是addvec函数地址，后续要通过这个位置实现函数的重定向，也就是说，当系统重定为完成之后，程序的执行流会继续回到这个位置，并且重新跳转到 .got.plt[5] 存储的地址中。由于那时候函数的地址已经是正确的地址了，所以可以正确运行。
```
0x555555557fe8: 0x0000000000003dd0
0x555555557ff0: 0x00007ffff7ffe2d0
0x555555557ff8: 0x00007ffff7fdd300
0x555555558000 <puts@got.plt>:  0x0000555555555036
0x555555558008 <printf@got.plt>:        0x0000555555555046
0x555555558010 <addvec@got.plt>:        0x0000555555555056
```

弄明白上面这个过程后，让我们接着看下面这两条指令。
```
(gdb) si
0x0000555555555056 in addvec@plt ()
(gdb) disass
Dump of assembler code for function addvec@plt:
   0x0000555555555050 <+0>:     jmp    *0x2fba(%rip)        # 0x555555558010 <addvec@got.plt>
=> 0x0000555555555056 <+6>:     push   $0x2
   0x000055555555505b <+11>:    jmp    0x555555555020
End of assembler dump.
```

push 2 是一条汇编指令，它将立即数值 2 压入堆栈。在这个上下文中，push 2 指令用于将 addvec 函数的 `.rela.plt` 节 索引压入堆栈。这样，在解析器被调用时，它就可以从堆栈中获取这个索引，并使用它来更新 addvec 函数在 `.got.plt` 中的地址。

下面这条命令查看目标程序的elf信息。

```bash
readelf -aW prog2l
```

```
重定位节 '.rela.plt' at offset 0x668 contains 3 entries:
    偏移量             信息             类型               符号值          符号名称 + 加数
0000000000004000  0000000300000007 R_X86_64_JUMP_SLOT     0000000000000000 puts@GLIBC_2.2.5 + 0
0000000000004008  0000000400000007 R_X86_64_JUMP_SLOT     0000000000000000 printf@GLIBC_2.2.5 + 0
0000000000004010  0000000500000007 R_X86_64_JUMP_SLOT     0000000000000000 addvec + 0
No processor specific unwind information to decode
```

还记得上文的`.plt`节吗？
```
0x0000555555555020 - 0x0000555555555060 is .plt
```
紧接着的`jmp`指令跳转到 0x0000555555555020 这个地址正式plt节的地址。另外，在elf信息中也已经说明了它们是可以被执行的，而`.got`等其他节是可以写的。

```
[12] .init             PROGBITS        0000000000001000 001000 00001b 00  AX  0   0  4
[13] .plt              PROGBITS        0000000000001020 001020 000040 10  AX  0   0 16
[14] .text             PROGBITS        0000000000001060 001060 000177 00  AX  0   0 16
[15] .fini             PROGBITS        00000000000011d8 0011d8 00000d 00  AX  0   0  4

......

[21] .dynamic          DYNAMIC         0000000000003dd0 002dd0 0001f0 10  WA  7   0  8
[22] .got              PROGBITS        0000000000003fc0 002fc0 000028 08  WA  0   0  8
[23] .got.plt          PROGBITS        0000000000003fe8 002fe8 000030 08  WA  0   0  8
[24] .data             PROGBITS        0000000000004018 003018 000020 00  WA  0   0  8
[25] .bss              NOBITS          0000000000004038 003038 000010 00  WA  0   0  8

......

Key to Flags:
  W (write), A (alloc), X (execute)
```
接着往下执行，程序执行流会接着跳转: `jmp 0x0000555555555020`

```
(gdb) disass 0x0000555555555020,0x000055555555502c
Dump of assembler code from 0x555555555020 to 0x55555555502c:
   0x0000555555555020:  push   0x2fca(%rip)        # 0x555555557ff0
   0x0000555555555026:  jmp    *0x2fcc(%rip)        # 0x555555557ff8
End of assembler dump.
```
系统将`.got.plt`节的第二项(.got.plt[1])压入到了栈中，为调用`_dl_fixup`函数做准备（后面会讲）。

如图所示(.got.plt节的内容上面有)：

```
(gdb) x /8gx $sp
0x7fffffffdd58: 0x00007ffff7ffe2d0      0x0000000000000002
```

接着执行
```
0x0000555555555026:  jmp    *0x2fcc(%rip)        # 0x555555557ff8
```
程序的执行流会跳转到`_dl_runtime_resolve_xsavec`函数。通过该函数的反汇编代码可以看到，它调用了 `_dl_fixup`函数，这个函数通过我们传入的参数来进行重定位工作。我们忽略函数`_dl_runtime_resolve_xsavec`，该函数功能就是保存寄存器的值到栈中，然后调用`_dl_fixup`执行具体的功能，从栈中恢复寄存器。
`_dl_fixup`函数传入的两个参数一个是rdi寄存器中存储的`link_map`，rsi是`.rela.plt`表中重定位的索引值，后面要根据该索引值写入新的地址。
```
0x00007ffff7fdd43d <+109>:   mov    0x10(%rbx),%rsi
0x00007ffff7fdd441 <+113>:   mov    0x8(%rbx),%rdi
0x00007ffff7fdd445 <+117>:   call   0x7ffff7fdb030 <_dl_fixup>
```

## 进入 _dl_fixup 函数

> 下文的许多宏和内联函数都可以在这个链接找到(https://codebrowser.dev/glibc/glibc/sysdeps/generic/ldsodefs.h.html#71)

[_dl_fixup](https://codebrowser.dev/glibc/glibc/elf/dl-runtime.c.html) 函数定义如下：
```c
DL_FIXUP_VALUE_TYPE
attribute_hidden __attribute ((noinline)) DL_ARCH_FIXUP_ATTRIBUTE
_dl_fixup (
# ifdef ELF_MACHINE_RUNTIME_FIXUP_ARGS
	   ELF_MACHINE_RUNTIME_FIXUP_ARGS,
# endif
	   struct link_map *l, ElfW(Word) reloc_arg)
```
传入的第一个参数 `0x00007ffff7ffe2d0` 指向了一个 [struct link_map](https://codebrowser.dev/glibc/glibc/include/link.h.html#link_map) 结构体。该结构体的定义下文给出了链接。使用gdb可以打印出来此时该结构体的内容。

```
(gdb) print /x *l
$1 = {l_addr = 0x555555554000, l_name = 0x7ffff7ffe890, l_ld = 0x555555557dd0, l_next = 0x7ffff7ffe8a0, 
  l_prev = 0x0, l_real = 0x7ffff7ffe2d0, l_ns = 0x0, l_libname = 0x7ffff7ffe878, l_info = {0x0, 0x555555557de0, 
    0x555555557ec0, 0x555555557eb0, 0x0, 0x555555557e60, 0x555555557e70, 0x555555557ef0, 0x555555557f00, 
    0x555555557f10, 0x555555557e80, 0x555555557e90, 0x555555557df0, 0x555555557e00, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 
    0x555555557ed0, 0x555555557ea0, 0x0, 0x555555557ee0, 0x0, 0x555555557e10, 0x555555557e30, 0x555555557e20, 
    0x555555557e40, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x555555557f40, 0x555555557f30, 0x0, 0x0, 
    0x555555557f20, 0x0, 0x555555557f60, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x555555557f50, 
    0x0 <repeats 25 times>, 0x555555557e50}, l_phdr = 0x555555554040, l_entry = 0x555555555060, l_phnum = 0xd, 
  l_ldnum = 0x0, l_searchlist = {r_list = 0x7ffff7fc3c48, r_nlist = 0x4}, l_symbolic_searchlist = {
    r_list = 0x7ffff7ffe870, r_nlist = 0x0}, l_loader = 0x0, l_versions = 0x7ffff7fc3c70, l_nversions = 0x4, 
  l_nbuckets = 0x1, l_gnu_bitmask_idxbits = 0x0, l_gnu_shift = 0x0, l_gnu_bitmask = 0x5555555543d0, {
    l_gnu_buckets = 0x5555555543d8, l_chain = 0x5555555543d8}, {l_gnu_chain_zero = 0x5555555543d8, 
    l_buckets = 0x5555555543d8}, l_direct_opencount = 0x1, l_type = 0x0, l_dt_relr_ref = 0x0, l_relocated = 0x1, 
  l_init_called = 0x1, l_global = 0x1, l_reserved = 0x0, l_main_map = 0x0, l_visited = 0x1, l_map_used = 0x1, 
  l_map_done = 0x0, l_phdr_allocated = 0x0, l_soname_added = 0x0, l_faked = 0x0, l_need_tls_init = 0x0, 
  l_auditing = 0x0, l_audit_any_plt = 0x0, l_removed = 0x0, l_contiguous = 0x1, l_free_initfini = 0x0, 
  l_ld_readonly = 0x0, l_find_object_processed = 0x0, l_nodelete_active = 0x0, l_nodelete_pending = 0x0, 
  l_property = 0x2, l_x86_feature_1_and = 0x0, l_x86_isa_1_needed = 0x1, l_1_needed = 0x0, l_rpath_dirs = {
    dirs = 0xffffffffffffffff, malloced = 0x0}, l_reloc_result = 0x0, l_versyms = 0x555555554562, l_origin = 0x0, 
  l_map_start = 0x555555554000, l_map_end = 0x555555558048, l_init_called_next = 0x7ffff7fc3150, l_scope_mem = {
    0x7ffff7ffe5a8, 0x0, 0x0, 0x0}, l_scope_max = 0x4, l_scope = 0x7ffff7ffe658, l_local_scope = {0x7ffff7ffe5a8, 
    0x0}, l_file_id = {dev = 0x0, ino = 0x0}, l_runpath_dirs = {dirs = 0xffffffffffffffff, malloced = 0x0}, 
  l_initfini = 0x7ffff7fc3c20, l_reldeps = 0x0, l_reldepsmax = 0x0, l_used = 0x1, l_feature_1 = 0x0, 
  l_flags_1 = 0x8000000, l_flags = 0x0, l_idx = 0x0, l_mach = {plt = 0x0, gotplt = 0x0, tlsdesc_table = 0x0}, 
  l_lookup_cache = {sym = 0x5555555544a0, type_class = 0x0, value = 0x7ffff7fc36d0, ret = 0x7ffff7ddaa90}, 
  l_tls_initimage = 0x0, l_tls_initimage_size = 0x0, l_tls_blocksize = 0x0, l_tls_align = 0x0, 
  l_tls_firstbyte_offset = 0x0, l_tls_offset = 0x0, l_tls_modid = 0x0, l_tls_dtor_count = 0x0, 
  l_relro_addr = 0x3dc0, l_relro_size = 0x240, l_serial = 0x0}
```

函数的前几句代码行为上都是类似的，所以，主要把精力放在第一句代码上。
```c
  const ElfW(Sym) *const symtab
    = (const void *) D_PTR (l, l_info[DT_SYMTAB]);
```
有两个陌生的宏，这两个宏在下文也多次出现。分别是`ElfW()` 和 `D_PTR`。

先来看这个 [ElfW()](https://codebrowser.dev/glibc/glibc/elf/link.h.html#30)
```c
/* We use this macro to refer to ELF types independent of the native wordsize.
   `ElfW(TYPE)' is used in place of `Elf32_TYPE' or `Elf64_TYPE'.  */
#define ElfW(type)	_ElfW (Elf, __ELF_NATIVE_CLASS, type)
#define _ElfW(e,w,t)	_ElfW_1 (e, w, _##t)
#define _ElfW_1(e,w,t)	e##w##t
```
[__ELF_NATIVE_CLASS](https://codebrowser.dev/glibc/glibc/bits/elfclass.h.html)
```c
#define __ELF_NATIVE_CLASS __WORDSIZE
```
[__WORDSIZE](https://codebrowser.dev/glibc/glibc/sysdeps/x86/bits/wordsize.h.html)
```c
/* Determine the wordsize from the preprocessor defines.  */
#if defined __x86_64__ && !defined __ILP32__
# define __WORDSIZE	64
#else
# define __WORDSIZE	32
#define __WORDSIZE32_SIZE_ULONG		0
#define __WORDSIZE32_PTRDIFF_LONG	0
#endif
```
根据定义和官方的注释，答案已经很明显了，ElfW 宏用于根据目标体系结构选择正确的 ELF 类型。而ElfW(Sym) 其实就是决定该使用哪种Sym结构体，其定义取决于目标架构。例如，在 32 位体系结构上，`ElfW(Sym)` 等同于 `Elf32_Sym`；而在 64 位体系结构上，`ElfW(Sym)` 等同于 `Elf64_Sym`。


[DT_SYMTAB](https://codebrowser.dev/glibc/glibc/elf/elf.h.html#861)

```c
/* Legal values for d_tag (dynamic entry type).  */
#define DT_NULL		0		/* Marks end of dynamic section */
#define DT_NEEDED	1		/* Name of needed library */
#define DT_PLTRELSZ	2		/* Size in bytes of PLT relocs */
#define DT_PLTGOT	3		/* Processor defined value */
#define DT_HASH		4		/* Address of symbol hash table */
#define DT_STRTAB	5		/* Address of string table */
#define DT_SYMTAB	6		/* Address of symbol table */
#define DT_RELA		7		/* Address of Rela relocs */
#define DT_RELASZ	8		/* Total size of Rela relocs */
#define DT_RELAENT	9		/* Size of one Rela reloc */
......
```
所以，l_info[DT_SYMTAB] 的值其实就是 `0x555555557e70` ，可以使用gdb查看下这个地址指向的结构体。
```
(gdb) p /x *l->l_info[6]
$6 = {
   d_tag = 0x6, 
   d_un = {
      d_val = 0x5555555543e0, 
      d_ptr = 0x5555555543e0
      }
   }
```
它指向了一个 `ElfW(Dyn)` 结构体，对于x64位，就是 Elf64_Dyn 结构体。而`.dynamic` 就是一个`ElfW(Dyn)` 结构体数组。

```c
typedef struct {
        Elf32_Sword d_tag;
        union {
                Elf32_Word      d_val;
                Elf32_Addr      d_ptr;
                Elf32_Off       d_off;
        } d_un;
} Elf32_Dyn;//32位程序

typedef struct {
        Elf64_Xword d_tag;
        union {
                Elf64_Xword     d_val;
                Elf64_Addr      d_ptr;
        } d_un;
} Elf64_Dyn;
//  在64位架构中，Elf64_Sxword 和 Elf64_Xword 都是8字节，
//  而 Elf64_Addr 也是8字节。因此，Elf64_Dyn 结构体的大小为16字节。
```
`d_ptr`：表示一个虚拟地址

`d_val`: 需要根据`d_tag` 才能决定表示的意思

`d_tag`：决定这个是什么类别的信息，以及如何解析 `d_un` 内部变量

[动态节 - 链接程序和库指南](https://docs.oracle.com/cd/E26926_01/html/E25910/chapter6-42444.html#scrolltoc) 介绍了所有的`d_tag` 类别，[英文版请点击这个](https://docs.oracle.com/cd/E23824_01/html/819-0690/chapter6-42444.html)

|名称|值|d_un|可执行文件|共享目标文件|
|:-:|:-:|:-:|:------:|:--------:|
|DT_STRTAB|5|d_ptr|强制|强制|
|DT_SYMTAB|6|d_ptr|强制|强制|

根据d_tag可以知道，链接器使用`d_un` 的 `d_ptr`。

在终端上输入 `readelf -dW prog2l` 可以查看`.dynamic`节的信息。
```
Dynamic section at offset 0x2dd0 contains 27 entries:
  标记        类型                         名称/值
 0x0000000000000001 (NEEDED)             共享库：[./libvector.so]
 0x0000000000000001 (NEEDED)             共享库：[libc.so.6]
 0x000000000000000c (INIT)               0x1000
 0x000000000000000d (FINI)               0x11d8
 0x0000000000000019 (INIT_ARRAY)         0x3dc0
 0x000000000000001b (INIT_ARRAYSZ)       8 (bytes)
 0x000000000000001a (FINI_ARRAY)         0x3dc8
 0x000000000000001c (FINI_ARRAYSZ)       8 (bytes)
 0x000000006ffffef5 (GNU_HASH)           0x3c0
 0x0000000000000005 (STRTAB)             0x4b8
 0x0000000000000006 (SYMTAB)             0x3e0
 0x000000000000000a (STRSZ)              170 (bytes)
 0x000000000000000b (SYMENT)             24 (bytes)
 0x0000000000000015 (DEBUG)              0x0
 0x0000000000000003 (PLTGOT)             0x3fe8
 0x0000000000000002 (PLTRELSZ)           72 (bytes)
 0x0000000000000014 (PLTREL)             RELA
 0x0000000000000017 (JMPREL)             0x668
 0x0000000000000007 (RELA)               0x5a8
 0x0000000000000008 (RELASZ)             192 (bytes)
 0x0000000000000009 (RELAENT)            24 (bytes)
 0x000000006ffffffb (FLAGS_1)            标志： PIE
 0x000000006ffffffe (VERNEED)            0x578
 0x000000006fffffff (VERNEEDNUM)         1
 0x000000006ffffff0 (VERSYM)             0x562
 0x000000006ffffff9 (RELACOUNT)          3
 0x0000000000000000 (NULL)               0x0

```

```
标记        类型                         名称/值
0x0000000000000006 (SYMTAB)             0x3e0
```
由于我们被加载到虚拟内存中的 `0x0000555555554000` 位置（页对齐）。将该值 + `0x3e0` 就跟我们在gdb上使用 `print /x *l->l_info[6]` 打印出来的结果一致。 


此时，让我们再回来看 [D_PTR](https://codebrowser.dev/glibc/glibc/sysdeps/generic/ldsodefs.h.html#89)
```c
/* All references to the value of l_info[DT_PLTGOT],
  l_info[DT_STRTAB], l_info[DT_SYMTAB], l_info[DT_RELA],
  l_info[DT_REL], l_info[DT_JMPREL], and l_info[VERSYMIDX (DT_VERSYM)]
  have to be accessed via the D_PTR macro.  The macro is needed since for
  most architectures the entry is already relocated - but for some not
  and we need to relocate at access time.  */
#define D_PTR(map, i) \
  ((map)->i->d_un.d_ptr + (dl_relocate_ld (map) ? 0 : (map)->l_addr))
```
```c
static inline bool dl_relocate_ld (const struct link_map *l)
{
        /* Don't relocate dynamic section if it is readonly  */
        return !(l->l_ld_readonly || DL_RO_DYN_SECTION);

}
```
gdb汇编的 `dl_relocate_ld` 函数的代码是这样的：
```
=> 0x00007ffff7fdb04d <+29>:    testb  $0x20,0x336(%rdi)
   0x00007ffff7fdb054 <+36>:    je     0x7ffff7fdb05c <_dl_fixup+44>
```
它会把 rdi+0x336 地址指向的值的最后一个字节与 0x20 进行 与 运算，他检查(char)*(0x336+0x7ffff7ffe2d0)的第6位是1或0，如果是1, AND操作数的结果就不会是0，那么ZF flag 将会被清除，如果是0，那么and操作数就会是0, ZF flag将会被设置。je 检查 ZF，如果ZF被设置，那么就执行跳转。
```
(gdb) p /x (char)*(0x336+0x7ffff7ffe2d0)
$15 = 0x8
```

现在，打印symtab的值就可以看到其实就是 (*l->l_info[6])->d_un.d_ptr。
```
(gdb) p symtab
$19 = (const Elf64_Sym * const) 0x5555555543e0

(gdb) p /x (*l->l_info[6])->d_un.d_ptr
$24 = 0x5555555543e0
```
现在就能理解这个宏的注释了。

> The macro is needed since for most architectures the entry is already relocated - but for some not and we need to relocate at access time.

对于我们的 x64 架构，它似乎并没有做什么，但对于其他架构，这是有必要的。通过查看 l->l_addr 的值就可以看出来，这个值正是程序被加载到虚拟内存页的起始地址`0x555555554000`，对于其他架构，需要把 .dynamic 节的偏移值 与 程序起始地址进行相加 重定位。

接下来这三句代码与第一行几乎一样，只不过是寻找到 `.dynamic` 节的 对应 的偏移值，然后加上 程序的起始地址。注意 reloc 变量，它加上了 `reloc_offset (pltgot, reloc_arg)`，需要注意的是 `reloc_arg` 其实就是 我们传入的 `.rela.plt` 节的偏移，其值为 2 。
```c
const char *strtab = (const void *) D_PTR (l, l_info[DT_STRTAB]);
const uintptr_t pltgot = (uintptr_t) D_PTR (l, l_info[DT_PLTGOT]);
const PLTREL *const reloc = (const void *) (D_PTR (l, l_info[DT_JMPREL])
        + reloc_offset (pltgot, reloc_arg));
```
下面是 `reloc_offset ()` 函数的定义。
```c

/* The ABI calls for the PLT stubs to pass the index of the relocation
   and not its offset.  In _dl_profile_fixup and _dl_audit_pltexit we
   also use the index.  Therefore it is wasteful to compute the offset
   in the trampoline just to reverse the operation immediately
   afterwards.  */
static inline uintptr_t
reloc_offset (uintptr_t plt0, uintptr_t pltn)
{
  return pltn * sizeof (ElfW(Rela));
}
```
经过上面的介绍，这个`ElfW(Rela)`相信不用说也就知道是 Elf64_Rela 了

`Elf64_Rela` 用于表示重定位表项。它对应与elf文件的 `.rela.plt` 节。它的定义如下：

```c
typedef struct {
    Elf64_Addr r_offset;  // Address
    Elf64_Xword r_info;   // Relocation type and symbol index
    Elf64_Sxword r_addend; // Addend
} Elf64_Rela;
```

每个成员的作用如下：

- `r_offset`：给出重定位所应用的位置。
- `r_info`：给出重定位类型和符号表索引。
- `r_addend`：给出常量加数，用于计算将被填充到可重定位字段中的值。

```
重定位节 '.rela.plt' at offset 0x668 contains 3 entries:
    偏移量             信息             类型               符号值          符号名称 + 加数
0000000000004000  0000000300000007 R_X86_64_JUMP_SLOT     0000000000000000 puts@GLIBC_2.2.5 + 0
0000000000004008  0000000400000007 R_X86_64_JUMP_SLOT     0000000000000000 printf@GLIBC_2.2.5 + 0
0000000000004010  0000000500000007 R_X86_64_JUMP_SLOT     0000000000000000 addvec + 0
```

此时 reloc 存储的值就是指向elf文件中 .rela.plt 节中 addvec 的位置。
```
(gdb) p /x *reloc
$46 = {r_offset = 0x4010, r_info = 0x500000007, r_addend = 0x0}
```


```c
const ElfW(Sym) *sym = &symtab[ELFW(R_SYM) (reloc->r_info)];
```

老实说，我自己第一次看这段代码也不知道做了什么，在结合了汇编后就豁然开朗了。

```
r12            0x555555554698       // reloc


   0x00007ffff7fdb080 <+80>:    mov    0x8(%r12),%rsi
   0x00007ffff7fdb085 <+85>:    mov    (%r12),%r13
   0x00007ffff7fdb089 <+89>:    mov    %rsi,%rax
=> 0x00007ffff7fdb08c <+92>:    add    %rdx,%r13
```

它把 reloc->r_info 的值复制到 %rsi 寄存器中，又把 reloc->r_offset 复制到 %r13 中。接着，
```asm
add     %rdx, %r13
```
由于 %rdx 存储的值正是程序的起始地址也就是 `l->l_addr` 的值 0x555555554000。加上 0x4010 后就成了 0x555555558010。这个值在上文中出现过，就是 .got.plt 节中 addvec 的地址。
```
(gdb) x /gx 0x555555558010
0x555555558010 <addvec@got.plt>:        0x0000555555555056
```
回顾一下，这个地址存储的值将来会被重新修改为addvec函数的正确的地址，延迟绑定会再次调用这个函数，那时候函数地址将会被重新修改为正确的地址，调用将会成功。

```
0x00007ffff7fdb08f <+95>:    shr    $0x20,%rax
0x00007ffff7fdb093 <+99>:    lea    (%rax,%rax,1),%rcx
0x00007ffff7fdb097 <+103>:   add    %rcx,%rax
0x00007ffff7fdb09a <+106>:   lea    (%r8,%rax,8),%rax
```

%rax 的值其实就是 0x500000007。右移之后就变成了 0x5。%r8 的值是 0x5555555543e0，对应变量symtab。那么这条lea指令目的就很明显了，就是将 0x5555555543e0 + (0x5) * 24 的值复制到 %rax中。获取`.dynsym` 节中 addvec的信息。

```
Symbol table '.dynsym' contains 9 entries:
   Num:    Value          Size Type    Bind   Vis      Ndx Name
     0: 0000000000000000     0 NOTYPE  LOCAL  DEFAULT  UND 
     1: 0000000000000000     0 FUNC    GLOBAL DEFAULT  UND __libc_start_main@GLIBC_2.34 (2)
     2: 0000000000000000     0 NOTYPE  WEAK   DEFAULT  UND _ITM_deregisterTMCloneTable
     3: 0000000000000000     0 FUNC    GLOBAL DEFAULT  UND puts@GLIBC_2.2.5 (3)
     4: 0000000000000000     0 FUNC    GLOBAL DEFAULT  UND printf@GLIBC_2.2.5 (3)
     5: 0000000000000000     0 FUNC    GLOBAL DEFAULT  UND addvec
     6: 0000000000000000     0 NOTYPE  WEAK   DEFAULT  UND __gmon_start__
     7: 0000000000000000     0 NOTYPE  WEAK   DEFAULT  UND _ITM_registerTMCloneTable
     8: 0000000000000000     0 FUNC    WEAK   DEFAULT  UND __cxa_finalize@GLIBC_2.2.5 (3)
```
可以看到 addvec在符号表的偏移就是 5 。对应与 `.rela.plt` 节中得到的值。

**符号表是由汇编器构造的，使用编译器输出到汇编语言.s文件的符号。.symtab 节中包含ELF符号表。这张符号表包含一个条目的数组。**

```c
typedef struct {
    Elf64_Word st_name; // 4 B (B for bytes)
    unsigned char st_info; // 1 B
    unsigned char st_other; // 1 B
    Elf64_Half st_shndx; // 2 B
    Elf64_Addr st_value; // 8 B
    Elf64_Xword st_size; // 8 B
} Elf64_Sym;
```

- `st_name` : .strtab 表中的字节偏移。
  
- `st_value`: 给出与符号相关联的数值，具体取值依赖于上下文，可能是一个正常的数值、一个地址等等。
  
  - 对于可重定位目标文件(xxx.o)，value是距定义目标的节的起始位置地址的偏移。
    
  - 对于可执行目标文件，value是一个绝对的运行时地址。
    
- `st_size`: 目标的大小(以字节为单位)。
  
- `st_info`: 给出符号的类型和绑定属性。
  
- `st_shndx` : 如果符号定义在该文件中，那么该成员为符号所在节 **在节区头部表中的下标**；如果符号不在本目标文件中，或者对于某些特殊的符号，该成员具有一些特殊含义。

`strtab + refsym->st_name` 的值就是 指向 `addvec` 字符串的起始地址。


***
# 先鸽了，抽空继续分析。
***
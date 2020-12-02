---
title: "Notes: Lazy binding"
date: 2020-12-02T12:08:22+08:00
draft: false
---

## Background

During the investigation of the ELF format, I found out that I was not familiar enough with the lazy binding mechanism. So in this article, I'll try to study and cover it with my notes and experiment. 

1. compile process

   ![](https://github.com/chuang76/Security-101/blob/master/05-linking/figure/proc.jpg?raw=true)

2. dynamic linking and lazy binding

   Why dynamic linking? (1) share codes, if the library (e.g. libc) contains a lot of functions, it is better to adopt code sharing mechanism. (2) modern systems typically use ASLR protection, so we should avoid hardcoding addresses. 

   What is lazy binding? Lazy binding means the symbolic references are not resolved until runtime. It ensures that the dynamic linker does not waste too much time on relocations. 

3. GOT and PLT sections 

   An ELF executable includes several sections. You can check section information with readelf command. To implement the lazy binding, we need two special sections called .got (.got.plt) and .plt. 

   Global offset table (GOT) is a table which contains absolute addresses of external symbols. GOT is represented as .got and .got.plt sections in an ELF file. A program can look up its GOT and extract the absolute value, then redirect the position-independent reference to the absolute location. The relocation entries (i.e., symbol to absolute address) in GOT are filled by the linker. 

   Procedure linkage table (PLT) is used to look up the addresses in GOT, and it is represented as .plt section in an ELF file. However, you probably would ask that why do we need PLT? It seems to be roundabout, but actually, it is necessary under a dynamic linked situation. The reason is at *compile* time, we have no idea about the absolute addresses of symbols.  So we need PLT as a stub to make an indirect jump. In this way, the original function calls can be unmodified. At runtime, the dynamic linker (ld.so) will be responsible for updating correct addresses. 

   

## Experiment 

Let's start with a simple example called p1.c. 

```c
#include <stdio.h>

int main(int argc, char** argv)
{
    puts("Hello world"); 
    puts("Hello world again"); 
    
    return 0; 
}
```

Compile it into an executable file with debugging information. 

```
$ gcc p1.c -no-pie -g -o p1
```

To understand how GOT and PLT work, we'll trace the program with gdb tool. First, disassemble the main function, and set a breakpoint at puts function (i.e., the address 0x401138). 

```
gdb-peda$ disas main
Dump of assembler code for function main:
   0x0000000000401122 <+0>:     push   rbp
   0x0000000000401123 <+1>:     mov    rbp,rsp
   0x0000000000401126 <+4>:     sub    rsp,0x10
   0x000000000040112a <+8>:     mov    DWORD PTR [rbp-0x4],edi
   0x000000000040112d <+11>:    mov    QWORD PTR [rbp-0x10],rsi
   0x0000000000401131 <+15>:    lea    rdi,[rip+0xecc]        # 0x402004
   0x0000000000401138 <+22>:    call   0x401030 <puts@plt>
   0x000000000040113d <+27>:    lea    rdi,[rip+0xecc]        # 0x402010
   0x0000000000401144 <+34>:    call   0x401030 <puts@plt>
   0x0000000000401149 <+39>:    mov    eax,0x0
   0x000000000040114e <+44>:    leave  
   0x000000000040114f <+45>:    ret    
End of assembler dump.
gdb-peda$ b *0x401138
Breakpoint 1 at 0x401138: file p1.c, line 5.
```

Run the program and check the current program counter to ensure we're going to call the puts function.

```
gdb-peda$ x/i $pc
=> 0x401138 <main+22>:  call   0x401030 <puts@plt>
```

Next, as you can see, we are trying to look up the GOT entry. 

```
gdb-peda$ x/i $pc
=> 0x401030 <puts@plt>: jmp    QWORD PTR [rip+0x2fe2]        # 0x404018 <puts@got.plt>
```

But the interesting thing is, the next instruction does nothing about GOT. Why? The reason is this is our first time to call puts function and our program adopts *lazy binding*. Hence, before resolving the reference, we can not obtain the actual address of puts function. So it goes back to .plt section. A slot number (0x0) is pushed onto the stack. It is a parameter for the resolution later.

```
gdb-peda$ x/2i $pc
=> 0x401036 <puts@plt+6>:       push   0x0
   0x40103b <puts@plt+11>:      jmp    0x401020
```

Now, it jumps to 0x401020, which is the location of a default stub in .plt section. Then, it pushes something pointed by 0x404008 onto the stack, and jumps to another location pointed by 0x404010.

```
gdb-peda$ x/2i $pc
=> 0x401020:    push   QWORD PTR [rip+0x2fe2]        # 0x404008
   0x401026:    jmp    QWORD PTR [rip+0x2fe4]        # 0x404010
```

Wait, what do the values pointed by 0x404008 and 0x404010? Let's check it. The first one is 0x00007ffff7ffe180, which is another argument for the resolution later. The second one is 0x00007ffff7fe8510. 

```
gdb-peda$ x/g 0x404008
0x404008:       0x00007ffff7ffe180
gdb-peda$ x/g 0x404010
0x404010:       0x00007ffff7fe8510
```

We can check the mapping information with vmmap command. As you can see, 0x00007ffff7fe8510 corresponds to ld-2.31.so., which means it belongs to the dynamic linker/loader. 

```
gdb-peda$ vmmap
Start              End                Perm      Name
...
0x00007ffff7fd2000 0x00007ffff7fd3000 r--p      /usr/lib/x86_64-linux-gnu/ld-2.31.so
0x00007ffff7fd3000 0x00007ffff7ff3000 r-xp      /usr/lib/x86_64-linux-gnu/ld-2.31.so
0x00007ffff7ff3000 0x00007ffff7ffb000 r--p      /usr/lib/x86_64-linux-gnu/ld-2.31.so
0x00007ffff7ffc000 0x00007ffff7ffd000 r--p      /usr/lib/x86_64-linux-gnu/ld-2.31.so
0x00007ffff7ffd000 0x00007ffff7ffe000 rw-p      /usr/lib/x86_64-linux-gnu/ld-2.31.so
0x00007ffff7ffe000 0x00007ffff7fff000 rw-p      mapped
0x00007ffffffde000 0x00007ffffffff000 rw-p      [stack]
```

Actually, its name is `_dl_runtime_resolve_xsave`.  `_dl_runtime_resolve_xsave` is a function served for the resolution. Basically, this function will use the parameters which we mentioned above to look up the symbol name "puts\0", use this string to find the actual address of puts function in the libraries (libc), then update the address in the GOT (.got.plt) entry. 

```
gdb-peda$ x/3i $pc
=> 0x7ffff7fe8510 <_dl_runtime_resolve_xsave>:          push   rbx
   0x7ffff7fe8511 <_dl_runtime_resolve_xsave+1>:        mov    rbx,rsp
   0x7ffff7fe8514 <_dl_runtime_resolve_xsave+4>:        and    rsp,0xffffffffffffffc0
```

As you can imagine, there is a long trip when we're calling a function provided by the libc library. But fortunately, after the actual address is resolved, we don't need to go through the whole resolution again. Let's disassemble the puts function again to verify it.

```
gdb-peda$ disas 'puts@plt'
Dump of assembler code for function puts@plt:
   0x0000000000401030 <+0>:     jmp    QWORD PTR [rip+0x2fe2]        # 0x404018 <puts@got.plt>
   0x0000000000401036 <+6>:     push   0x0
   0x000000000040103b <+11>:    jmp    0x401020
End of assembler dump.
```

The actual address in the .got.plt entry is updated as 0x00007ffff7e635b0, which is located in the libc library. You can also set another breakpoint at 0x401144 (call the puts function again) and trace the program to verify the idea.

```
gdb-peda$ x/g 0x404018
0x404018 <puts@got.plt>:        0x00007ffff7e635b0

gdb-peda$ xinfo 0x00007ffff7e635b0
0x7ffff7e635b0 (<__GI__IO_puts>:        push   r14)
Virtual memory mapping:
Start : 0x00007ffff7e12000
End   : 0x00007ffff7f5d000
Offset: 0x515b0
Perm  : r-xp
Name  : /usr/lib/x86_64-linux-gnu/libc-2.31.so
```



## Reference

1. Practical Binary Analysis, Dennis Andriesse. 
2. GOT and PLT for pwning. [[link]](https://systemoverlord.com/2017/03/19/got-and-plt-for-pwning.html)
3. Understanding _dl_runtime_resolve(). [[link]](https://ypl.coffee/dl-resolve/)


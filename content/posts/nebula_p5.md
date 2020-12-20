---
title: "Writeup: Nebula (level 6)"
date: 2020-12-19T16:05:18+08:00
draft: false
---

Nebula is a virtual machine that covers a variety of challenges about Linux privilege escalation, scripting language issues, and file system race conditions. It is available at [Exploit Exercises](https://exploit-exercises.lains.space/).

<br>

## Level 6

The description of Level 6 is provided as follows.

```
The flag06 account credentials came from a legacy unix system.

To do this level, log in as the level06 account with the password level06. 
Files for this level can be found in /home/flag06.
```

If you are not familiar with the password file (/etc/passwd) and shadow file (/etc/shadow) in the Unix-based system, you can check this [post](https://chuang76.github.io/posts/users_and_groups/) I wrote before. In this challenge, we are going to use the password file to get the flag. 

Let's check the contents of /etc/passwd first. The second field is about password information, i.e., encrypted password. In the modern system, the second field is typically stored as `x`, which means the actual content is maintained in another file called /etc/shadow.

![](https://github.com/chuang76/image/blob/master/06-1.PNG?raw=true)

However, this is a legacy Unix system, the password is not "hidden" in the shadow file. So we can use [John The Ripper](https://zh.wikipedia.org/wiki/John_The_Ripper), a free crafting passwords tool to decode the password. 

```
$ cat passwd
flag06:ueqwOCnSGdsuM:993:993::/home/flag06:/bin/sh

$ john passwd
Loaded 1 password hash (descrypt, traditional crypt(3) [DES 128/128 SSE2-16])
Press 'q' or Ctrl-C to abort, almost any other key for status
hello            (flag06)
1g 0:00:00:00 100% 2/3 20.00g/s 15060p/s 15060c/s 15060C/s 123456..marley
Use the "--show" option to display all of the cracked passwords reliably
Session completed
```

Now, we have the decoded password, i.e., "hello". Let's use it to log in as account flag06 and get the flag. 

![](https://github.com/chuang76/image/blob/master/06-2.PNG?raw=true)




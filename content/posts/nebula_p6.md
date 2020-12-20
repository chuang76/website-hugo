---
title: "Writeup: Nebula (level 7)"
date: 2020-12-19T16:09:16+08:00
draft: true
---

Nebula is a virtual machine that covers a variety of challenges about Linux privilege escalation, scripting language issues, and file system race conditions. It is available at [Exploit Exercises](https://exploit-exercises.lains.space/).

<br>

## Level 7

Here is the description of Level 6.

```
The flag07 user was writing their very first perl program that allowed them to 
ping hosts to see if they were reachable from the web server.

To do this level, log in as the level07 account with the password level07. 
Files for this level can be found in /home/flag07.
```

The contents in /home/flag07 can be displayed as follows. 

![](https://github.com/chuang76/image/blob/master/07-1.PNG?raw=true)

Let's view the contents of the file index.cgi. It is a CGI script written in Perl. [CGI](https://en.wikipedia.org/wiki/Common_Gateway_Interface) (Common Gateway Interface) is a specification for web servers to execute programs like command-line running on a server that generates dynamic web pages. 

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

Subsequently, let's view the contents of the file thttpd.conf. It is a configure file of thttpd. Here is the part of thttpd.conf.

```
port=7007
dir=/home/flag07
user=flag07
cgipat=**.cgi
```


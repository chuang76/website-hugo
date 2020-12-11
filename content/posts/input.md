---
title: "Writeup: input"
date: 2020-12-09T15:00:11+08:00
draft: false
---

This is a challenge from [pwnable.kr](https://pwnable.kr/). After connecting to the remote server with ssh command (or downloading the files with scp command), we can view the source code as follows. As you can see, our goal is to clear all of the stages and capture the flag. 

```
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <sys/socket.h>
#include <arpa/inet.h>

int main(int argc, char* argv[], char* envp[]){
    printf("Welcome to pwnable.kr\n");
    printf("Let's see if you know how to give input to program\n");
    printf("Just give me correct inputs then you will get the flag :)\n");

    // argv
    if(argc != 100) return 0;
    if(strcmp(argv['A'],"\x00")) return 0;
    if(strcmp(argv['B'],"\x20\x0a\x0d")) return 0;
    printf("Stage 1 clear!\n");

    // stdio
    char buf[4];
    read(0, buf, 4);
    if(memcmp(buf, "\x00\x0a\x00\xff", 4)) return 0;
    read(2, buf, 4);
    if(memcmp(buf, "\x00\x0a\x02\xff", 4)) return 0;
    printf("Stage 2 clear!\n");

    // env
    if(strcmp("\xca\xfe\xba\xbe", getenv("\xde\xad\xbe\xef"))) return 0;
    printf("Stage 3 clear!\n");

    // file
    FILE* fp = fopen("\x0a", "r");
    if(!fp) return 0;
    if( fread(buf, 4, 1, fp)!=1 ) return 0;
    if( memcmp(buf, "\x00\x00\x00\x00", 4) ) return 0;
    fclose(fp);
    printf("Stage 4 clear!\n");

    // network
    int sd, cd;
    struct sockaddr_in saddr, caddr;
    sd = socket(AF_INET, SOCK_STREAM, 0);
    if(sd == -1){
        printf("socket error, tell admin\n");
        return 0;
    }
    saddr.sin_family = AF_INET;
    saddr.sin_addr.s_addr = INADDR_ANY;
    saddr.sin_port = htons( atoi(argv['C']) );
    if(bind(sd, (struct sockaddr*)&saddr, sizeof(saddr)) < 0){
    	printf("bind error, use another port\n");
        return 1;
    }
    listen(sd, 1);
    int c = sizeof(struct sockaddr_in);
    cd = accept(sd, (struct sockaddr *)&caddr, (socklen_t*)&c);
    if(cd < 0){
        printf("accept error, tell admin\n");
        return 0;
    }
    if( recv(cd, buf, 4, 0) != 4 ) return 0;
    if(memcmp(buf, "\xde\xad\xbe\xef", 4)) return 0;
    printf("Stage 5 clear!\n");

    // here's your flag
    system("/bin/cat flag");
    return 0;
}
```



## Stage 1: execve()

Stage 1 requires us to provide 100 arguments. Apart from that, the 65th ('A') and 66th ('B') arguments should be "\x00" and "\x20\x0a\x0d", respectively. We can use the function `execve()` to execute the program referred by the pathname and specify the arguments and the environment. The prototype of `execve()` is as follows. Note that both argv and envp must be terminated by a NULL pointer.

```
#include <unistd.h>
int execve(const char *pathname, char *const argv[], char *const envp[]);
```

Here is a short example of `execve()`, it can list the contents of current directory. 

```
#include <unistd.h>

int main(int argc, char** argv)
{
        char* args[] = {"ls", "-al", NULL};
        char* env[] = {NULL};
        execve("/bin/ls", args, env);
        return 0;
}
```

Hence, according to the source code, we can write the payload to clear stage 1 as follows. 

```
char* args[101] = {0};
char* env[] = {NULL};

for (int i = 0; i < 100; i++)
	args[i] = "a";

args['A'] = "\x00";
args['B'] = "\x20\x0a\x0d";
args[100] = NULL;

execve("/home/input2/input", args, env);
```

<br>

## Stage 2: Pipes and I/O Redirection

Next, the program reads something from the standard input (namely stdin, whose file descriptor is 0) into a buffer, which should be "\x00\x0a\x00\xff". Again, the program reads something from the standard error (stderr, whose file descriptor is 2) into the buffer, which should be "\x00\x0a\x02\xff". We can use the concept of pipes and I/O redirection to clear this stage. 

Pipes are used to redirect a stream from one program to another. In the Unix-based systems, `pipe()` function is provided to create the pipes. It takes an array of two integers as a argument, pipefd[0] is set up for reading, pipefd[1] is set up for writing. Therefore, if a process is going to write something to another process, the first one integer, pipefd[0], should be disabled. On the other hand, another process that is going to read something should disable the second one integer, i.e., pipefd[1].

```
#include <unistd.h>
int pipe(int pipefd[2]);
```

 Here is a short example of `pipe()`: parent process receives data "Hello world" from the child process. 

```
#include <unistd.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

int main(int argc, char** argv)
{
        int     fd[2], nbytes;
        pid_t   pid;
        char    str[] = "Hello world\n";
        char    buf[80];

        pipe(fd);				// create pipes

        pid = fork();

        if (pid == -1) {
                perror("fork");
                exit(1);
        }

        if (pid == 0) {
                // child process: write
                sleep(1);
                close(fd[0]);
                write(fd[1], str, strlen(str) + 1);
                exit(0);
        }
        else {
                // parent process: read
                close(fd[1]);
                nbytes = read(fd[0], buf, sizeof(buf));
                printf("Received: %s", buf);
        }

        return 0;
}
```

Besides, we can use `dup2()` system call to duplicate the input descriptor of the pipe onto the stdin or stderr. 

```
int fd_stdin[2], fd_stderr[2];
pid_t pid;                      // child process pid
pipe(fd_stdin);                 // create pipes
pipe(fd_stderr);

pid = fork();
if (pid == -1) {
	perror("fork"); 
	exit(1); 
}
if (pid == 0) {
    close(fd_stdin[0]);
    write(fd_stdin[1], "\x00\x0a\x00\xff", 4);     

    close(fd_stderr[0]);
    write(fd_stderr[1], "\x00\x0a\x02\xff", 4);
}
else {
    close(fd_stdin[1]);
    dup2(fd_stdin[0], 0);       // (old fd, new fd)

    close(fd_stderr[1]);
    dup2(fd_stderr[0], 2);
}
```

<br>

## Stage 3: Environment variable

Stage 3 is about the environment variable. As we mentioned above (at stage 1), we can use execve() to set up the environment. Note that the name of the variable is "xde\xad\xbe\xef" while the value is "\xca\xfe\xba\xbe". Do not mix up. 

```
char* env[] = {"\xde\xad\xbe\xef=\xca\xfe\xba\xbe", NULL};
```

<br>

## Stage 4: fopen() and fwrite()

Let's move on. At stage 4, the program reads a file called "x0a", and the first 4 bytes of the file is "\x00\x00\x00\x00". Thus, we can just create a file and write the string into this specified file. 

```
FILE* fp;
fp = fopen("\x0a", "wb");
if (fp == NULL) {
    perror("fopen");
    exit(1);
}
fwrite("\x00\x00\x00\x00", 4, 1, fp);
fclose(fp);
```

<br>

## Stage 5: Socket 

The last one, stage 5, is about socket programming. The program creates a socket, whose domain is AP_INET (IPv4) and the type is SOCK_STREAM (TCP). After the bind and accept operation, it receives the data "\xde\xad\xbe\xef" from a certain port. The port number is specified by the argv['C']. To respond to this, we should create a socket (as a client), then send the data to the target process. Again, the port number, which is argv['C'] in this case, should be assigned with execve() function. 

Hence, we can combine 5 parts into the payload called `exp.c` as follow. 

```
#include <unistd.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <sys/types.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>

int main(int argc, char** argv)
{
        // stage 1: execve
        char* args[101] = {0};
        char* env[] = {"\xde\xad\xbe\xef=\xca\xfe\xba\xbe", NULL};    

        for (int i = 0; i < 100; i++)
                args[i] = "a";  

        args['A'] = "\x00";
        args['B'] = "\x20\x0a\x0d";
        args['C'] = "9527";
        args[100] = NULL;

        // execve("/home/input2/input", args, env);

        // stage 2: I/O redirection
        int     fd_stdin[2], fd_stderr[2];         // file descriptor 
        pid_t   pid;                               // child process pid
        pipe(fd_stdin);
        pipe(fd_stderr);

        // stage 4: create a file, write
        FILE*   fp;
        fp = fopen("\x0a", "wb");
        if (fp == NULL) {
                perror("fopen");
                exit(1);
        }
        fwrite("\x00\x00\x00\x00", 4, 1, fp);
        fclose(fp);

        pid = fork();
        if (pid == -1) {
                perror("fork");
                exit(1);
        }

        if (pid == 0) {
                // child: write
                close(fd_stdin[0]);
                write(fd_stdin[1], "\x00\x0a\x00\xff", 4);

                close(fd_stderr[0]);
                write(fd_stderr[1], "\x00\x0a\x02\xff", 4);
        }
        else {
                // parent: read
                close(fd_stdin[1]);
                dup2(fd_stdin[0], 0);           // (old fd, new fd)

                close(fd_stderr[1]);
                dup2(fd_stderr[0], 2);

                execve("/home/input2/input", args, env);
        }
        
        // stage 5: socket
        int sd;
        sd = socket(AF_INET, SOCK_STREAM, 0);
        if (sd == -1) {
                perror("socket");
                exit(1);
        }

        struct sockaddr_in saddr;
        bzero(&saddr, sizeof(saddr));
        saddr.sin_family = AF_INET;
        saddr.sin_addr.s_addr = inet_addr("127.0.0.1");    // local host
        saddr.sin_port = htons(9527);

        int ret = connect(sd, (struct sockaddr *)&saddr, sizeof(saddr));

        if (ret == -1) {
                perror("connect");
                exit(1);
        }

        // send data
        send(sd, "\xde\xad\xbe\xef", 4, 0);
        close(sd);   
        return 0;
}
```

However, we don't have the permission to create or write the program under `/home/input2`. So we need to create a directory under `/tmp`, then upload or create our exploit program in this directory. Apart from that, we should create a symbolic link to reference the original executable flag. 

```
input2@pwnable:/tmp/rax$ ln -sf /home/input2/flag flag 

input2@pwnable:/tmp/rax$ ls -al
total 514476
drwxrwxr-x     2 input2 input2      4096 Dec 11 08:50 .
drwxrwx-wt 18306 root   root   526786560 Dec 11 08:59 ..
-rwxrwxr-x     1 input2 input2     13544 Dec  9 04:00 exp
-rwxrwxr-x     1 input2 input2      1866 Dec  9 04:00 exp.c
lrwxrwxrwx     1 input2 input2        17 Dec  9 03:59 flag -> /home/input2/flag
```

Here is the flag. 

```
input2@pwnable:/tmp/rax$ ./exp
Welcome to pwnable.kr
Let's see if you know how to give input to program
Just give me correct inputs then you will get the flag :)
Stage 1 clear!
Stage 2 clear!
Stage 3 clear!
Stage 4 clear!
Stage 5 clear!
Mommy! I learned how to pass various input in Linux :)
```




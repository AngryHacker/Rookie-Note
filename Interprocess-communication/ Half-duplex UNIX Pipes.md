### 管道概述及相关API应用
#### 管道相关的关键概念
管道是 Linux 支持的最初 Unix IPC 形式之一，具有以下特点：

* 管道是半双工的，数据只能向一个方向流动；需要双方通信时，需要建立起两个管道；
* 只能用于父子进程或者兄弟进程之间（具有亲缘关系的进程）；
* 单独构成一种独立的文件系统：管道对于管道两端的进程而言，就是一个文件，但它不是普通的文件，它不属于某种文件系统，而是自立门户，单独构成一种文件系统，并且只存在与内存中。
* 数据的读出和写入：一个进程向管道中写的内容被管道另一端的进程读出。写入的内容每次都添加在管道缓冲区的末尾，并且每次都是从缓冲区的头部读出数据。

#### 管道的创建

```c
#include <unistd.h>
int pipe(int fd[2])
```
该函数创建的管道的两端处于一个进程中间，在实际应用中没有太大意义，因此，一个进程在由 pipe() 创建管道后，一般再 fork 一个子进程，然后通过管道实现父子进程间的通信（因此也不难推出，只要两个进程中存在亲缘关系，这里的亲缘关系指的是具有共同的祖先，都可以采用管道方式来进行通信）。

#### 管道的读写规则
管道两端可分别用描述字 fd[0] 以及 fd[1] 来描述，需要注意的是，管道的两端是固定了任务的。即一端只能用于读，由描述字 fd[0] 表示，称其为管道读端；另一端则只能用于写，由描述字 fd[1] 来表示，称其为管道写端。如果试图从管道写端读取数据，或者向管道读端写入数据都将导致错误发生。一般文件的 I/O 函数都可以用于管道，如 close、read、write 等等。

##### 从管道中读取数据：
* 如果管道的写端不存在，则认为已经读到了数据的末尾，读函数返回的读出字节数为 0；
* 当管道的写端存在时，如果请求的字节数目大于 `PIPE_BUF`，则返回管道中现有的数据字节数，如果请求的字节数目不大于 `PIPE_BUF`，则返回管道中现有数据字节数（此时，管道中数据量小于请求的数据量）；或者返回请求的字节数（此时，管道中数据量不小于请求的数据量）。注：（`PIPE_BUF` 在`include/linux/limits.h` 中定义，不同的内核版本可能会有所不同。Posix.1 要求 PIPE_BUF 至少为512 字节，red hat 7.2 中为 4096）。

###### 验证 `PIPE_BUF` 大小：

```c
#include <unistd.h>
#include <stdint.h>
#include <stdio.h>

int main() {
    uint32_t pipe_buf_size = pathconf("PIPE_BUF", _PC_PIPE_BUF);
    printf("The size of PIPE_BUF is: %d\n", pipe_buf_size);
    return 0;
}
```

###### 关于管道的读规则验证：

```c
/* 验证写端关闭后数据一直存在，直到被读出 */
#include <unistd.h>
#include <string.h>
#include <stdio.h>
#include <stdlib.h>

int main()
{
    int   pipe_fd[2];
    pid_t pid;
    char  r_buf[100];
    char  w_buf[4];
    char* p_wbuf;
    int   r_num;
    int   cmd;

    memset(r_buf, 0, sizeof(r_buf));
    memset(w_buf, 0, sizeof(r_buf));
    p_wbuf = w_buf;
    if (pipe(pipe_fd) < 0)
    {
        printf("pipe create error\n");
        return -1;
    }

    if ((pid = fork()) == 0)
    {
        printf("\n");
        close(pipe_fd[1]);
        /* 确保父进程关闭写端 */
        sleep(3);
        r_num = read(pipe_fd[0], r_buf, 100);
        printf(	"read num is %d, the data read from the pipe is %d\n", r_num, atoi(r_buf));
        close(pipe_fd[0]);
        exit(0);
    } else if (pid > 0) {
        /* 关闭读端 */
        close(pipe_fd[0]);
        strcpy(w_buf,"111");
        if (write(pipe_fd[1], w_buf, 4) != -1)
            printf("parent write over\n");

        /* 关闭写端 */
        close(pipe_fd[1]);

        printf("parent close fd[1] over\n");

        sleep(10);
    }
}
```

输出结果：

```shell
➜  ipc  gcc -o pipe1 pipe1.c
➜  ipc  ./pipe1             
parent write over
parent close fd[1] over

read num is 4, the data read from the pipe is 111
```

##### 向管道中写入数据：

* 向管道中写入数据大于 `PIPE_BUF`（管道缓冲区大小） 时，linux 将不保证写入的原子性，这里的原子性指的是写操作的数据可能相互穿插。
* 如果读进程不读走管道缓冲区中的数据，那么写操作将一直阻塞。 

注：只有在管道的读端存在时，向管道中写入数据才有意义。否则，向管道中写入数据的进程将收到内核传来的 `SIFPIPE` 信号，应用程序可以处理该信号，也可以忽略（默认动作则是应用程序终止）。

###### 验证写端对读端存在的依赖性：

```c
#include <unistd.h>
#include <string.h>
#include <stdio.h>
#include <stdlib.h>
#include <signal.h>

int main()
{
    /* 捕捉管道断裂产生的 SIGPIPE 信号 */
    signal(SIGPIPE, SIG_IGN);

    int   pipe_fd[2];
    pid_t pid;
    char  r_buf[4];
    char* w_buf;
    int   writenum;
    int   cmd;

    memset(r_buf, 0, sizeof(r_buf));
    if (pipe(pipe_fd) < 0)
    {
        printf("pipe create error\n");
        return -1;
    }
    if ((pid = fork()) == 0) {
        close(pipe_fd[0]);
        close(pipe_fd[1]);
        sleep(10);
        exit(0);
    } else if (pid > 0) {
        sleep(1);  /* 等待子进程完成关闭读端的操作 */
        close(pipe_fd[0]);
        w_buf = "111";
        if ((writenum = write(pipe_fd[1], w_buf, 4)) == -1) {
            printf("write to pipe error\n");
        } else {
            printf("the bytes write to pipe is %d \n", writenum);
            close(pipe_fd[1]);
        }
    }
    return 0;
}
```

输出结果：

```shell
➜  ipc  gcc -o pipe2 pipe2.c
➜  ipc  ./pipe2             
write to pipe error
```

分析：管道的所有读端都被关闭后，依然向管道写，会产生 SIGPIPE 信号，系统默认是终止应用程序，这样就看不到错误是什么。上面程序加了 `signal(SIGPIPE, SIG_IGN);` 是捕捉该信号，并交给系统处理，这样会产生 write to pipe error 的提示。。因此，在向管道写入数据时，至少应该存在某一个进程，其中管道读端没有被关闭，否则就会出现上述错误。

###### 验证写入管道的原子性：

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <sys/types.h>
#include <errno.h>
#include <fcntl.h>


#define ERR_EXIT(m) \
        do \
        { \
                perror(m); \
                exit(EXIT_FAILURE); \
        } while(0)

#define TEST_SIZE 68*1024

int main(void)
{
    char a[TEST_SIZE];
    char b[TEST_SIZE];
    char c[TEST_SIZE];

    memset(a, 'A', sizeof(a));
    memset(b, 'B', sizeof(b));
    memset(c, 'C', sizeof(c));

    int pipefd[2];

    int ret = pipe(pipefd);
    if (ret == -1)
        ERR_EXIT("pipe error");

    pid_t pid;
    pid = fork();
    if (pid == 0)//第一个子进程
    {
        close(pipefd[0]);
        ret = write(pipefd[1], a, sizeof(a));
        printf("process a: pid=%d write %d bytes to pipe\n", getpid(), ret);
        exit(0);
    }

    pid = fork();


    if (pid == 0)//第二个子进程
    {
        close(pipefd[0]);
        ret = write(pipefd[1], b, sizeof(b));
        printf("process b: pid=%d write %d bytes to pipe\n", getpid(), ret);
        exit(0);
    }

    pid = fork();


    if (pid == 0)//第三个子进程
    {
        close(pipefd[0]);
        ret = write(pipefd[1], c, sizeof(c));
        printf("process c: pid=%d write %d bytes to pipe\n", getpid(), ret);
        exit(0);
    }


    close(pipefd[1]);

    sleep(1);
    int fd = open("test.txt", O_WRONLY | O_CREAT | O_TRUNC, 0644);
    /* 缓冲区大小 */
    char buf[1024*4] = {0};
    int n = 1;
    while (1)
    {
        ret = read(pipefd[0], buf, sizeof(buf));
        if (ret == 0)
            break;
        printf("n=%02d pid=%d read %d bytes from pipe buf[4095]=%c\n", n++, getpid(), ret, buf[4095]);
        write(fd, buf, ret);

    }
    return 0;
}
```

输出结果：

```
➜  ipc  ./pipe3
n=01 pid=23164 read 4096 bytes from pipe buf[4095]=C
n=02 pid=23164 read 4096 bytes from pipe buf[4095]=C
process c: pid=23167 write 69632 bytes to pipe
n=03 pid=23164 read 4096 bytes from pipe buf[4095]=C
n=04 pid=23164 read 4096 bytes from pipe buf[4095]=C
n=05 pid=23164 read 4096 bytes from pipe buf[4095]=C
n=06 pid=23164 read 4096 bytes from pipe buf[4095]=C
n=07 pid=23164 read 4096 bytes from pipe buf[4095]=C
n=08 pid=23164 read 4096 bytes from pipe buf[4095]=C
n=09 pid=23164 read 4096 bytes from pipe buf[4095]=C
n=10 pid=23164 read 4096 bytes from pipe buf[4095]=C
n=11 pid=23164 read 4096 bytes from pipe buf[4095]=C
n=12 pid=23164 read 4096 bytes from pipe buf[4095]=C
n=13 pid=23164 read 4096 bytes from pipe buf[4095]=C
n=14 pid=23164 read 4096 bytes from pipe buf[4095]=C
n=15 pid=23164 read 4096 bytes from pipe buf[4095]=C
n=16 pid=23164 read 4096 bytes from pipe buf[4095]=C
n=17 pid=23164 read 4096 bytes from pipe buf[4095]=A
n=18 pid=23164 read 4096 bytes from pipe buf[4095]=C
n=19 pid=23164 read 4096 bytes from pipe buf[4095]=B
n=20 pid=23164 read 4096 bytes from pipe buf[4095]=B
n=21 pid=23164 read 4096 bytes from pipe buf[4095]=A
n=22 pid=23164 read 4096 bytes from pipe buf[4095]=B
n=23 pid=23164 read 4096 bytes from pipe buf[4095]=A
n=24 pid=23164 read 4096 bytes from pipe buf[4095]=B
n=25 pid=23164 read 4096 bytes from pipe buf[4095]=A
n=26 pid=23164 read 4096 bytes from pipe buf[4095]=B
n=27 pid=23164 read 4096 bytes from pipe buf[4095]=A
n=28 pid=23164 read 4096 bytes from pipe buf[4095]=B
n=29 pid=23164 read 4096 bytes from pipe buf[4095]=A
n=30 pid=23164 read 4096 bytes from pipe buf[4095]=B
n=31 pid=23164 read 4096 bytes from pipe buf[4095]=A
n=32 pid=23164 read 4096 bytes from pipe buf[4095]=B
n=33 pid=23164 read 4096 bytes from pipe buf[4095]=A
process b: pid=23166 write 69632 bytes to pipe
n=34 pid=23164 read 4096 bytes from pipe buf[4095]=B
process a: pid=23165 write 69632 bytes to pipe
n=35 pid=23164 read 4096 bytes from pipe buf[4095]=A
n=36 pid=23164 read 4096 bytes from pipe buf[4095]=B
n=37 pid=23164 read 4096 bytes from pipe buf[4095]=A
n=38 pid=23164 read 4096 bytes from pipe buf[4095]=B
n=39 pid=23164 read 4096 bytes from pipe buf[4095]=A
n=40 pid=23164 read 4096 bytes from pipe buf[4095]=B
n=41 pid=23164 read 4096 bytes from pipe buf[4095]=A
n=42 pid=23164 read 4096 bytes from pipe buf[4095]=B
n=43 pid=23164 read 4096 bytes from pipe buf[4095]=A
n=44 pid=23164 read 4096 bytes from pipe buf[4095]=B
n=45 pid=23164 read 4096 bytes from pipe buf[4095]=A
n=46 pid=23164 read 4096 bytes from pipe buf[4095]=B
n=47 pid=23164 read 4096 bytes from pipe buf[4095]=A
n=48 pid=23164 read 4096 bytes from pipe buf[4095]=B
n=49 pid=23164 read 4096 bytes from pipe buf[4095]=A
n=50 pid=23164 read 4096 bytes from pipe buf[4095]=B
n=51 pid=23164 read 4096 bytes from pipe buf[4095]=A
```

分析：这里是开启了三个子进程，分别向管道写入 68×1024 大小的数据，远远大于管道缓冲区 PIPE_BUF，子进程 a 写入的数据都是字符 a，子进程 b 写入的数据都是字符 b，子进程 c 写入的数据都是字符 c。如果管道能保证原子性的话，那么写操作不会穿插进行，保证一种字符写完后才会出现另一种字符，但这里看到的是三种字符 abc 在过程中交叉出现，如果你打开运行目录下生成的 test.txt，也可以看到这一点。

#### 管道应用实例

##### 实例一：用于shell
管道可用于输入输出重定向，它将一个命令的输出直接定向到另一个命令的输入。比如，当在某个 shell 程序（Bourne shell 或 C shell 等）键入 who│wc -l 后，相应 shell 程序将创建 who 以及 wc 两个进程和这两个进程间的管道。

```shell
➜  ipc  who
rookie   :0           2016-01-08 09:24 (:0)
rookie   pts/1        2016-01-08 09:26 (ubuntu)
rookie   pts/19       2016-01-08 15:15 (ubuntu)
rookie   pts/20       2016-01-08 15:15 (ubuntu)
➜  ipc  who | wc -l
4
```

##### 实例二：用于具有亲缘关系的进程间通信
下面例子给出了管道的具体应用，父进程通过管道发送一些命令给子进程，子进程解析命令，并根据命令作相应处理。

```c
#include <unistd.h>
#include <sys/types.h>
#include <stdio.h>
#include <string.h>
#include <stdlib.h>

int handle_cmd(int);

int main() {
    int    pipe_fd[2];
    pid_t  pid;
    char   r_buf[4];
    char*  w_buf[256];
    int    childexit = 0;
    int    i, cmd;

    memset(r_buf, 0, sizeof(r_buf));
    if (pipe(pipe_fd) < 0) {
        printf("pipe create error\n");
        return -1;
    }

    /* 子进程：解析从管道中获取的命令，并作相应的处理 */
    if ((pid = fork()) == 0) {
        printf("\n");
        close(pipe_fd[1]);
        sleep(2);

        while (!childexit) {
            read(pipe_fd[0], r_buf, 4);
            cmd = atoi(r_buf);
            if(cmd == 0) {
                printf("child: receive command from parent over\n now child process exit\n");
                childexit = 1;
            } else if (handle_cmd(cmd) != 0) {
                printf("child end\n");
                return -1;
            }
            sleep(1);
        }
        close(pipe_fd[0]);
        exit(0);
    } else if (pid > 0) {
        /* 父进程发送命令 */
        close(pipe_fd[0]);
        w_buf[0] = "003";
        w_buf[1] = "005";
        w_buf[2] = "777";
        w_buf[3] = "000";
        for(i = 0;i < 4;i++)
            write(pipe_fd[1], w_buf[i], 4);
        close(pipe_fd[1]);
        exit(0);
    }
}

//下面是子进程的命令处理函数（特定于应用）
int handle_cmd(int cmd) {
    /* cmd 应该在 0 到 256 之间 */
    if ((cmd < 0) || (cmd > 256)) {
        printf("child: invalid command \n");
        return -1;
    }
    printf("child: the cmd from parent is %d\n", cmd);
    return 0;
}
```

#### 管道的局限性
管道的主要局限性正体现在它的特点上：

* 只支持单向数据流
* 只能用于具有亲缘关系的进程之间
* 没有名字
* 管道的缓冲区是有限的（管道制存在于内存中，在管道创建时，为缓冲区分配一个页面大小）
* 管道所传送的是无格式字节流，这就要求管道的读出方和写入方必须事先约定好数据的格式，比如多少字节算作一个消息（或命令、或记录）等等

整理来自：[Reference1](http://www.ibm.com/developerworks/cn/linux/l-ipc/part1/index.html#authorN10019)
[Reference2](http://www.cnblogs.com/mickole/p/3192461.html)

（注：主要来自 Reference1, 但原文的程序示例中有大量错误，所以进行了不少修改和替换。如有问题，请指正。）

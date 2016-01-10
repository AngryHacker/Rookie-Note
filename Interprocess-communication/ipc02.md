### 有名管道概述及相关API应用
#### 有名管道相关的关键概念
管道应用的一个重大限制是它没有名字，因此，只能用于具有亲缘关系的进程间通信，在有名管道（named pipe 或 FIFO）提出后，该限制得到了克服。FIFO 不同于管道之处在于它提供一个路径名与之关联，以 FIFO 的文件形式存在于文件系统中。这样，即使与 FIFO 的创建进程不存在亲缘关系的进程，只要可以访问该路径，就能够彼此通过 FIFO 相互通信（能够访问该路径的进程以及 FIFO 的创建进程之间），因此，通过 FIFO 不相关的进程也能交换数据。值得注意的是，FIFO 严格遵循先进先出（first in first out），对管道及 FIFO 的读总是从开始处返回数据，对它们的写则把数据添加到末尾。它们不支持诸如 `lseek()` 等文件定位操作。

#### 有名管道的创建

```c
#include <sys/types.h>
#include <sys/stat.h>
int mkfifo(const char * pathname, mode_t mode)
```

该函数的第一个参数是一个普通的路径名，也就是创建后 FIFO 的名字。第二个参数与打开普通文件的`open()` 函数中的 mode 参数相同。 如果 mkfifo 的第一个参数是一个已经存在的路径名时，会返回 EEXIST 错误，所以一般典型的调用代码首先会检查是否返回该错误，如果确实返回该错误，那么只要调用打开 FIFO 的函数就可以了。一般文件的 I/O 函数都可以用于 FIFO，如 close、read、write 等等。

#### 有名管道的打开规则
有名管道比管道多了一个打开操作：open。

FIFO的打开规则：

如果当前打开操作是为读而打开 FIFO 时，若已经有相应进程为写而打开该 FIFO，则当前打开操作将成功返回；否则，可能阻塞直到有相应进程为写而打开该 FIFO（当前打开操作设置了阻塞标志）；或者，成功返回（当前打开操作没有设置阻塞标志）。

如果当前打开操作是为写而打开 FIFO 时，如果已经有相应进程为读而打开该 FIFO，则当前打开操作将成功返回；否则，可能阻塞直到有相应进程为读而打开该 FIFO（当前打开操作设置了阻塞标志）；或者，返回 ENXIO 错误（当前打开操作没有设置阻塞标志）。

#### 有名管道的读写规则
##### 从 FIFO 中读取数据：

约定：如果一个进程为了从 FIFO 中读取数据而阻塞打开 FIFO，那么称该进程内的读操作为设置了阻塞标志的读操作。

* 如果有进程写打开 FIFO，且当前 FIFO 内没有数据，则对于设置了阻塞标志的读操作来说，将一直阻塞。对于没有设置阻塞标志读操作来说则返回-1，当前 errno 值为 EAGAIN，提醒以后再试。
* 对于设置了阻塞标志的读操作说，造成阻塞的原因有两种：当前 FIFO 内有数据，但有其它进程在读这些数据；另外就是 FIFO 内没有数据。解阻塞的原因则是 FIFO 中有新的数据写入，不论信写入数据量的大小，也不论读操作请求多少数据量。
* 读打开的阻塞标志只对本进程第一个读操作施加作用，如果本进程内有多个读操作序列，则在第一个读操作被唤醒并完成读操作后，其它将要执行的读操作将不再阻塞，即使在执行读操作时，FIFO 中没有数据也一样（此时，读操作返回0）。
* 如果没有进程写打开 FIFO，则设置了阻塞标志的读操作会阻塞。

注：如果 FIFO 中有数据，则设置了阻塞标志的读操作不会因为 FIFO 中的字节数小于请求读的字节数而阻塞，此时，读操作会返回 FIFO 中现有的数据量。

##### 向FIFO中写入数据：

约定：如果一个进程为了向 FIFO 中写入数据而阻塞打开 FIFO，那么称该进程内的写操作为设置了阻塞标志的写操作。

对于设置了阻塞标志的写操作：
* 当要写入的数据量不大于 `PIPE_BUF` 时，linux 将保证写入的原子性。如果此时管道空闲缓冲区不足以容纳要写入的字节数，则进入睡眠，直到当缓冲区中能够容纳要写入的字节数时，才开始进行一次性写操作。
* 当要写入的数据量大于 `PIPE_BUF` 时，linux 将不再保证写入的原子性。FIFO 缓冲区一有空闲区域，写进程就会试图向管道写入数据，写操作在写完所有请求写的数据后返回。

对于没有设置阻塞标志的写操作：
* 当要写入的数据量大于 `PIPE_BUF` 时，linux 将不再保证写入的原子性。在写满所有 FIFO 空闲缓冲区后，写操作返回。
* 当要写入的数据量不大于 `PIPE_BUF` 时，linux 将保证写入的原子性。如果当前 FIFO 空闲缓冲区能够容纳请求写入的字节数，写完后成功返回；如果当前 FIFO 空闲缓冲区不能够容纳请求写入的字节数，则返回 EAGAIN 错误，提醒以后再写。

#### 应用示例

##### 生产者程序

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <fcntl.h>
#include <limits.h>
#include <sys/types.h>
#include <sys/stat.h>

#define FIFO_NAME "/tmp/Linux/my_fifo"
#define BUFFER_SIZE PIPE_BUF
#define TEN_MEG (1024 * 1024 * 10)

int main()
{
    int pipe_fd;
    int res;
    int open_mode = O_WRONLY;

    int bytes = 0;
    char buffer[BUFFER_SIZE + 1];

    if (access(FIFO_NAME, F_OK) == -1)
    {
        res = mkfifo(FIFO_NAME, 0777);
        if (res != 0)
        {
            fprintf(stderr, "Could not create fifo %s/n", FIFO_NAME);
            exit(EXIT_FAILURE);
        }
    }

    printf("Process %d opening FIFO O_WRONLY/n", getpid());
    pipe_fd = open(FIFO_NAME, open_mode);
    printf("Process %d result %d/n", getpid(), pipe_fd);

    if (pipe_fd != -1)
    {
        while (bytes < TEN_MEG)
        {
            res = write(pipe_fd, buffer, BUFFER_SIZE);
            if (res == -1)
            {
                fprintf(stderr, "Write error on pipe/n");
                exit(EXIT_FAILURE);
            }
            bytes += res;
        }
        close(pipe_fd);
    }
    else
    {
        exit(EXIT_FAILURE);
    }

    printf("Process %d finish/n", getpid());
    exit(EXIT_SUCCESS);
}
```

##### 消费者程序

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <fcntl.h>
#include <limits.h>
#include <sys/types.h>
#include <sys/stat.h>

#define FIFO_NAME "/tmp/Linux/my_fifo"
#define BUFFER_SIZE PIPE_BUF

int main()
{
    int pipe_fd;
    int res;

    int open_mode = O_RDONLY;
    char buffer[BUFFER_SIZE + 1];
    int bytes = 0;

    memset(buffer, '/0', sizeof(buffer));

    printf("Process %d opeining FIFO O_RDONLY/n", getpid());
    pipe_fd = open(FIFO_NAME, open_mode);
    printf("Process %d result %d/n", getpid(), pipe_fd);

    if (pipe_fd != -1)
    {
        do{
            res = read(pipe_fd, buffer, BUFFER_SIZE);
            bytes += res;
        }while(res > 0);
        close(pipe_fd);
    }
    else
    {
        exit(EXIT_FAILURE);
    }

    printf("Process %d finished, %d bytes read/n", getpid(), bytes);
    exit(EXIT_SUCCESS);
}
```

### 小结

管道常用于两个方面：

 1. 在 shell 中时常会用到管道（作为输入输入的重定向），在这种应用方式下，管道的创建对于用户来说是透明的；
 2. 用于具有亲缘关系的进程间通信，用户自己创建管道，并完成读写操作。

* FIFO 可以说是管道的推广，克服了管道无名字的限制，使得无亲缘关系的进程同样可以采用先进先出的通信机制进行通信。
* 管道和 FIFO 的数据是字节流，应用程序之间必须事先确定特定的传输"协议"，采用传播具有特定意义的消息。
* 要灵活应用管道及 FIFO，理解它们的读写规则是关键。

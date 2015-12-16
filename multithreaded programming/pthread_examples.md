### 多线程创建
#### 参考源码
```c
#include <stdio.h>
#include <pthread.h>
#include <string.h>
#include <sys/types.h>
#include <unistd.h>

pthread_t child_tid;

void print_ids(const char *str) {
    pid_t pid; /* 进程 id */
    pthread_t tid; /* 线程 id */
    pid = getpid(); /* 获取进程 id */
    tid = pthread_self(); /* 获取线程 id */
    printf("%s pid: %u, tid: %u (0x%x)\n",
            str,
            (unsigned int)pid,
            (unsigned int)tid,
            (unsigned int)tid);
}

void *func(void *arg) {
    print_ids("new thread:");
    return ((void *)0);
}

int main(int argc, char **argv) {
     int err;
     err = pthread_create(&child_tid, NULL, func, NULL); /* 创建线程 */
     if (err != 0) {
         printf("create thread error: %s\n", strerror(err));
         return 1;
     } else {
         printf("create thread success! new thread id: %u\n", (unsigned int)child_tid);
     }
     print_ids("main thread:");
     sleep(1);
     return 0;
}
```

#### 运行结果

```shell
➜  pthread  gcc -Wall -o pthread_create pthread_create.c -pthread
➜  pthread  ./pthread_create 
create thread success! new thread id: 2954086144
main thread: pid: 3214, tid: 2962421504 (0xb092f700)
new thread: pid: 3214, tid: 2954086144 (0xb013c700)
```

### 多线程条件变量
#### 参考源码

```c
#include <stdio.h>
#include <pthread.h>
#include <unistd.h>

pthread_mutex_t counter_lock;   /* 互斥锁 */
pthread_cond_t counter_nonzero; /* 条件变量 */
int counter = 0;
int estatus = -1;

void *decrement_counter(void *argv);
void *increment_counter(void *argv);

int main(int argc, char **argv) {
    printf("counter: %d\n", counter);
    pthread_t thd1, thd2;
    int ret;

    /* 初始化 */
    pthread_mutex_init(&counter_lock, NULL);
    pthread_cond_init(&counter_nonzero, NULL);

    ret = pthread_create(&thd1, NULL, decrement_counter, NULL); /* 创建线程1 */
    if (ret) {
        perror("del:\n");
        return 1;
    }

    sleep(1);  /* 注：相比原文添加这个使得运行结果对条件变量更好理解 */

    ret = pthread_create(&thd2, NULL, increment_counter, NULL); /* 创建线程2 */
    if (ret) {
        perror("inc: \n");
        return 1;
    }

    int counter = 0;
    while (counter != 10) {
        printf("counter(main): %d\n", counter); /* 主线程 */
        sleep(1);
        counter++;
    }

    pthread_exit(0);

    return 0;
}

void *decrement_counter(void *argv) {
    printf("counter(decrement): %d\n", counter);
    pthread_mutex_lock(&counter_lock);
    while (counter == 0)
        pthread_cond_wait(&counter_nonzero, &counter_lock); /* 进入阻塞(wait)，等待激活(signal) */

    printf("counter--(before): %d\n", counter);
    counter--; /* 等待signal激活后再执行 */
    printf("counter--(after): %d\n", counter);
    pthread_mutex_unlock(&counter_lock);

    return &estatus;
}

void *increment_counter(void *argv) {
    printf("counter(increment): %d\n", counter);
    pthread_mutex_lock(&counter_lock);
    if (counter == 0) {
        /* 激活(signal)阻塞(wait)的线程(先执行完signal线程，然后再执行wait线程) */
        pthread_cond_signal(&counter_nonzero); 
    }

    printf("counter++(before): %d\n", counter);
    counter++;
    printf("counter++(after): %d\n", counter);
    pthread_mutex_unlock(&counter_lock);

    return &estatus;
}
```

#### 运行结果

```shell
➜  pthread  gcc -pthread -Wall -o pthread_cond pthread_cond.c
➜  pthread  ./pthread_cond 
counter: 0
counter(decrement): 0
counter(main): 0
counter(increment): 0
counter++(before): 0
counter++(after): 1
counter--(before): 1
counter--(after): 0
counter(main): 1
counter(main): 2
counter(main): 3
counter(main): 4
counter(main): 5
counter(main): 6
counter(main): 7
counter(main): 8
counter(main): 9
```

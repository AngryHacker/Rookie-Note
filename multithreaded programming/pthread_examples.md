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
### 线程私有数据（TSD）
#### 参考源码

```c
/**
 * 线程中特有的线程存储， Thread Specific Data 。在多线程程序中，所有线程共享程序中的全局变量。
 * 现在有一全局变量，所有线程都可以使用它，改变它的值。而如果每个线程希望能单独拥有它，
 * 那么就需要使用线程存储了。表面上看起来这是一个全局变量，所有线程都可以使用它，
 * 而它的值在每一个线程中又是单独存储的。这就是线程存储的意义。
 *
 * 下面说一下线程存储的具体用法。
 *
 *   创建一个类型为 pthread_key_t 类型的变量。
 *
 *   调用 pthread_key_create() 来创建该变量。该函数有两个参数，
 *   第一个参数就是上面声明的 pthread_key_t 变量，第二个参数是一个清理函数，用来在线程释放该线程存储的时候被调用。
 *   该函数指针可以设成 NULL ，这样系统将调用默认的清理函数。
 *
 *   当线程中需要存储特殊值的时候，可以调用 pthread_setspcific() 。该函数有两个参数，
 *   第一个为前面声明的 pthread_key_t 变量，第二个为 void* 变量，这样你可以存储任何类型的值。
 *
 *   如果需要取出所存储的值，调用 pthread_getspecific() 。
 *   该函数的参数为前面提到的 pthread_key_t 变量，该函数返回 void * 类型的值。
 *
 * 下面是前面提到的函数的原型：
 *
 * int pthread_setspecific(pthread_key_t key, const void *value);
 *
 * void *pthread_getspecific(pthread_key_t key);
 *
 * int pthread_key_create(pthread_key_t *key, void (*destructor)(void*));
 */

#include <stdio.h>
#include <pthread.h>
#include <unistd.h>

pthread_key_t key; /* 声明参数key */

void echomsg(void *arg) /* 析构处理函数 */
{
    printf("destruct executed in thread = %u, arg = %p\n",
            (unsigned int)pthread_self(),
            arg);
}

void *child_1(void *arg)
{
    pthread_t tid;

    tid = pthread_self();
    printf("%s: thread %u enter\n", (char *)arg, (unsigned int)tid);

    pthread_setspecific(key, (void *)tid);  /* 与key值绑定的value(tid) */
    printf("%s: thread %u returns %p\n",    /* %p 表示输出指针格式  */
            (char *)arg,
            (unsigned int)tid,
            pthread_getspecific(key));  /* 获取key值的value */
    sleep(1);
    return NULL;
}

void *child_2(void *arg)
{
    pthread_t tid;

    tid = pthread_self();
    printf("%s: thread %u enter\n", (char *)arg, (unsigned int)tid);

    pthread_setspecific(key, (void *)tid);
    printf("%s: thread %u returns %p\n",
            (char *)arg,
            (unsigned int)tid,
            pthread_getspecific(key));
    sleep(1);
    return NULL;
}

int main(void)
{
    pthread_t tid1, tid2;

    printf("hello main\n");

    pthread_key_create(&key, echomsg); /* 创建key */

    pthread_create(&tid1, NULL, child_1, (void *)"child_1"); /* 创建带参数的线程，需要强制转换 */
    pthread_create(&tid2, NULL, child_2, (void *)"child_2");

    sleep(3);
    pthread_key_delete(key); /* 清除key */
    printf("bye main\n");

    pthread_exit(0);
    return 0;
}
```

#### 运行结果

```
➜  pthread  gcc -Wall -pthread -o pthread_setspecific pthread_setspecific.c 
➜  pthread  ./pthread_setspecific 
hello main
child_2: thread 4238526208 enter
child_2: thread 4238526208 returns 0x7fdafca2c700
child_1: thread 4246918912 enter
child_1: thread 4246918912 returns 0x7fdafd22d700
destruct executed in thread = 4238526208, arg = 0x7fdafca2c700
destruct executed in thread = 4246918912, arg = 0x7fdafd22d700
bye main
```

### `pthread_once`
#### 参考源码

```c
#include <stdio.h>
#include <pthread.h>
#include <unistd.h>

pthread_once_t once = PTHREAD_ONCE_INIT; /* 声明变量。控制变量必须使用 PTHREAD_ONCE_INIT 宏静态地初始化 */

/**
 * once_run()函数仅执行一次，且究竟在哪个线程中执行是不定的
 * 尽管pthread_once(&once,once_run)出现在两个线程中
 * 函数原型：int pthread_once(pthread_once_t *once_control, void (*init_routine)(void))
 */
void once_run(void)
{
    printf("Func: %s in thread: %u\n",
            __func__,
            (unsigned int)pthread_self());
}

void *child_1(void *arg)
{
    pthread_t tid;

    tid = pthread_self();
    pthread_once(&once, once_run); /* 调用once_run */
    printf("%s: thread %d returns\n", (char *)arg, (unsigned int)tid);

    return NULL;
}

void *child_2(void *arg)
{
    pthread_t tid;

    tid = pthread_self();
    pthread_once(&once, once_run); /* 调用once_run */
    printf("%s: thread %d returns\n", (char *)arg, (unsigned int)tid);

    return NULL;
}

int main(void)
{
    pthread_t tid1, tid2;

    printf("hello main\n");
    pthread_create(&tid1, NULL, child_1, (void *)"child_1");
    pthread_create(&tid2, NULL, child_2, (void *)"child_2");

    pthread_join(tid1, NULL);  /* main主线程等待线程 tid1 返回 */
    pthread_join(tid2, NULL);  /* main主线程等待线程 tid2 返回 */
    printf("bye main\n");

    return 0;
}
```

#### 运行结果

```
➜  pthread  gcc -Wall -pthread -o pthread_once pthread_once.c
➜  pthread  ./pthread_once                                 
hello main
Func: once_run in thread: 861472512
child_2: thread 861472512 returns
child_1: thread 869865216 returns
bye main
```

整理来自：[Reference1](http://blog.csdn.net/sunboy_2050/article/details/6063067) [Reference2](http://www.cnblogs.com/yuxingfirst/archive/2012/07/25/2608612.html)
（注：整理过程不同程度修改了代码，全部已在 gcc 5.2.1 编译通过）

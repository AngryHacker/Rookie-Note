### 互斥锁创建
有两种方法创建互斥锁，静态方式和动态方式。

#### 静态
POSIX 定义了一个宏 PTHREAD_MUTEX_INITIALIZER 来静态初始化互斥锁，方法如下：

```c
pthread_mutex_t mutex = PTHREAD_MUTEX_INITIALIZER;
```

在 LinuxThreads 实现中，pthread_mutex_t是一个结构，而 PTHREAD_MUTEX_INITIALIZER 则是一个结构常量。

#### 动态
动态方式是采用pthread_mutex_init()函数来初始化互斥锁，API定义如下：

```c
int pthread_mutex_init(pthread_mutex_t *mutex, const pthread_mutexattr_t *mutexattr)
```

其 中mutexattr 用于指定互斥锁属性（见下），如果为 NULL 则使用缺省属性。

`pthread_mutex_destroy()` 用于注销一个互斥锁，API 定义如下：

```c
int pthread_mutex_destroy(pthread_mutex_t *mutex)
```
销毁一个互斥锁即意味着释放它所占用的资源，且要求锁当前处于开放状态。由于在 Linux 中，互斥锁并不占用任何资源，因此 LinuxThreads 中的 `pthread_mutex_destroy()` 除了检查锁状态以外（锁定状态则返回 EBUSY）没有其他动作。

### 互斥锁属性
互斥锁的属性在创建锁的时候指定，在 LinuxThreads 实现中仅有一个锁类型属性，不同的锁类型在试图对一个已经被锁定的互斥锁加锁时表现不同。当前（glibc2.2.3,linuxthreads0.9）有四个值可供选择：
* `PTHREAD_MUTEX_TIMED_NP`，这是缺省值，也就是普通锁。当一个线程加锁以后，其余请求锁的线程将形成一个等待队列，并在解锁后按优先级获得锁。这种锁策略保证了资源分配的公平性。
* `PTHREAD_MUTEX_RECURSIVE_NP`，嵌套锁，允许同一个线程对同一个锁成功获得多次，并通过多次 unlock解锁。如果是不同线程请求，则在加锁线程解锁时重新竞争。
* `PTHREAD_MUTEX_ERRORCHECK_NP`，检错锁，如果同一个线程请求同一个锁，则返回 EDEADLK，否则与 PTHREAD_MUTEX_TIMED_NP 类型动作相同。这样就保证当不允许多次加锁时不会出现最简单情况下的死锁。
* `PTHREAD_MUTEX_ADAPTIVE_NP`，适应锁，动作最简单的锁类型，仅等待解锁后重新竞争。

### 互斥锁操作

锁操作主要包括加锁 `pthread_mutex_lock()`、解锁 `pthread_mutex_unlock()` 和测试加锁 `pthread_mutex_trylock()` 三个，不论哪种类型的锁，都不可能被两个不同的线程同时得到，而必须等待解锁。对于普通锁和适应锁类型，解锁者可以是同进程内任何线程；而检错锁则必须由加锁者解锁才有效，否则返回 EPERM；对于嵌套锁，文档和实现要求必须由加锁者解锁，但实验结果表明并没有这种限制，这个不同目前还没有得到解释。在同一进程中的线程，如果加锁后没有解锁，则任何其他线程都无法再获得锁。

```c
int pthread_mutex_lock(pthread_mutex_t *mutex)
int pthread_mutex_unlock(pthread_mutex_t *mutex)
int pthread_mutex_trylock(pthread_mutex_t *mutex)
```

`pthread_mutex_trylock()` 语义与 `pthread_mutex_lock()` 类似，不同的是在锁已经被占据时返回 EBUSY 而不是挂起等待。

### 例子
#### Example 1

```c
#include <stdio.h>
#include <stdlib.h>
#include <pthread.h>
#include <unistd.h>

void *function(void *arg);
pthread_mutex_t mutex;
int counter = 0;

int main(int argc, char *argv[]) {
    int rc1, rc2;

    char *str1 = "weloveocean";
    char *str2 = "daydaystudy";
    pthread_t thread1, thread2;

    pthread_mutex_init(&mutex, NULL);
    if ((rc1 = pthread_create(&thread1, NULL, function, str1)) != 0) {
        fprintf(stdout, "thread 1 create failed: %d\n", rc1);
    }

    if ((rc2 = pthread_create(&thread2, NULL, function, str2)) != 0) {
        fprintf(stdout, "thread 2 create failed: %d\n", rc2);
    }

    pthread_join(thread1, NULL);
    pthread_join(thread2, NULL);
    return 0;
}

void *function(void *arg)
{
    char *m;
    m = (char *)arg;
    pthread_mutex_lock(&mutex);
    while (*m != '\0') {
        printf("%c",*m);
        fflush(stdout);
        m++;
        sleep(1);
    }
    printf("\n");
    pthread_mutex_unlock(&mutex);
    return (void*)0;
}
```

##### 运行结果

```
➜  pthread  gcc -Wall -pthread -o pthread_mutex1 pthread_mutex1.c 
➜  pthread  ./pthread_mutex1                                    
daydaystudy
weloveocean
```

#### Example2

```c
#include<stdlib.h>
#include<stdio.h>
#include<unistd.h>
#include<pthread.h>

typedef struct ct_sum {
    int sum;
    pthread_mutex_t lock;
}ct_sum;

void * add1(void * cnt)
{
    pthread_mutex_lock(&(((ct_sum*)cnt)->lock));
    int i;
    for (i = 0;i < 50;i++) {
        (*(ct_sum*)cnt).sum += i;
    }
    pthread_mutex_unlock(&(((ct_sum*)cnt)->lock));
    pthread_exit(NULL);
    return 0;
}

void * add2(void *cnt) {
    int i;
    cnt = (ct_sum*)cnt;
    pthread_mutex_lock(&(((ct_sum*)cnt)->lock));
    for (i = 50;i < 101;i++) {
        (*(ct_sum*)cnt).sum += i;
    }
    pthread_mutex_unlock(&(((ct_sum*)cnt)->lock));
    pthread_exit(NULL);
    return 0;
}

int main(void) {
    pthread_t ptid1, ptid2;
    ct_sum cnt;
    pthread_mutex_init(&(cnt.lock), NULL);
    cnt.sum = 0;
    pthread_create(&ptid1, NULL, add1, &cnt);
    pthread_create(&ptid2, NULL, add2, &cnt);

    pthread_join(ptid1, NULL);
    pthread_join(ptid2, NULL);

    printf("sum %d\n", cnt.sum);
    pthread_mutex_destroy(&(cnt.lock));
    return 0;
}
```

##### 运行结果

```
➜  pthread  gcc -Wall -pthread -o pthread_mutex2 pthread_mutex2.c 
➜  pthread  ./pthread_mutex2                                    
sum 5050
```

整理来自：[Reference1](http://blog.csdn.net/yusiguyuan/article/details/14148311)  [Reference2](http://blog.csdn.net/wypblog/article/details/7264315)

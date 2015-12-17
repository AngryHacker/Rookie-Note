### 初始化条件变量 `pthread_cond_init`

```c
#include <pthread.h>
int pthread_cond_init(pthread_cond_t *cv, const pthread_condattr_t *cattr);
```

返回值：函数成功返回 0；任何其他返回值都表示错误

初始化一个条件变量。当参数 cattr 为空指针时，函数创建的是一个缺省的条件变量。否则条件变量的属性将由 cattr 中的属性值来决定。调用 `pthread_cond_init` 函数时，参数 cattr 为空指针等价于 cattr 中的属性为缺省属性，只是前者不需要 cattr 所占用的内存开销。这个函数返回时，条件变量被存放在参数 cv 指向的内存中。

可以用宏 `PTHREAD_COND_INITIALIZER` 来初始化静态定义的条件变量，使其具有缺省属性。这和用 `pthread_cond_init` 函数动态分配的效果是一样的。初始化时不进行错误检查。如：

```c
pthread_cond_t cv = PTHREAD_COND_INITIALIZER;
```
不能由多个线程同时初始化一个条件变量。当需要重新初始化或释放一个条件变量时，应用程序必须保证这个条件变量未被使用。
 
### 阻塞在条件变量上 `pthread_cond_wait`

```c
#include <pthread.h>
int pthread_cond_wait(pthread_cond_t *cv, pthread_mutex_t *mutex);
```

返回值：函数成功返回 0；任何其他返回值都表示错误

函数将解锁 mutex 参数指向的互斥锁，并使当前线程阻塞在 cv 参数指向的条件变量上。

被阻塞的线程可以被 `pthread_cond_signal` 函数、`pthread_cond_broadcast` 函数唤醒，也可能在被信号中断后被唤醒。

`pthread_cond_wait` 函数的返回并不意味着条件的值一定发生了变化，必须重新检查条件的值。

`pthread_cond_wait` 函数返回时，相应的互斥锁将被当前线程锁定，即使是函数出错返回。

一般一个条件表达式都是在一个互斥锁的保护下被检查。当条件表达式未被满足时，线程将仍然阻塞在这个条件变量上。当另一个线程改变了条件的值并向条件变量发出信号时，等待在这个条件变量上的一个线程或所有线程被唤醒，接着都试图再次占有相应的互斥锁。

阻塞在条件变量上的线程被唤醒以后，直到 `pthread_cond_wait()` 函数返回之前条件的值都有可能发生变化。所以函数返回以后，在锁定相应的互斥锁之前，必须重新测试条件值。最好的测试方法是循环调用 `pthread_cond_wait` 函数，并把满足条件的表达式置为循环的终止条件。如：

```c
pthread_mutex_lock();
while (condition_is_false)
 pthread_cond_wait();
pthread_mutex_unlock();
```

阻塞在同一个条件变量上的不同线程被释放的次序是不一定的。

注意：`pthread_cond_wait()` 函数是退出点，如果在调用这个函数时，已有一个挂起的退出请求，且线程允许退出，这个线程将被终止并开始执行善后处理函数，而这时和条件变量相关的互斥锁仍将处在锁定状态。
 
### 解除在条件变量上的阻塞 `pthread_cond_signal`

```c
#include <pthread.h>
int pthread_cond_signal(pthread_cond_t *cv);
```

返回值：函数成功返回 0；任何其他返回值都表示错误

函数被用来释放被阻塞在指定条件变量上的一个线程。

必须在互斥锁的保护下使用相应的条件变量。否则对条件变量的解锁有可能发生在锁定条件变量之前，从而造成死锁。

唤醒阻塞在条件变量上的所有线程的顺序由调度策略决定，如果线程的调度策略是 SCHED_OTHER 类型的，系统将根据线程的优先级唤醒线程。

如果没有线程被阻塞在条件变量上，那么调用 `pthread_cond_signal()` 将没有作用。
 
### 阻塞直到指定时间 `pthread_cond_timedwait`

```c
#include <pthread.h>
#include <time.h>
int pthread_cond_timedwait(pthread_cond_t *cv, pthread_mutex_t *mp, const structtimespec * abstime);
```

返回值：函数成功返回 0；任何其他返回值都表示错误

函数到了一定的时间，即使条件未发生也会解除阻塞。这个时间由参数 abstime 指定。函数返回时，相应的互斥锁往往是锁定的，即使是函数出错返回。

注意：`pthread_cond_timedwait` 函数也是退出点。

超时时间参数是指一天中的某个时刻。使用举例：

```c
pthread_timestruc_t to;
to.tv_sec = time(NULL) + TIMEOUT;
to.tv_nsec = 0;
```
超时返回的错误码是 `ETIMEDOUT`。
 
### 释放阻塞的所有线程 `pthread_cond_broadcast`

```c
#include <pthread.h>
int pthread_cond_broadcast(pthread_cond_t *cv);
```

返回值：函数成功返回 0；任何其他返回值都表示错误

函数唤醒所有被 `pthread_cond_wait` 函数阻塞在某个条件变量上的线程，参数 cv 被用来指定这个条件变量。当没有线程阻塞在这个条件变量上时，`pthread_cond_broadcast` 函数无效。

由于 `pthread_cond_broadcast` 函数唤醒所有阻塞在某个条件变量上的线程，这些线程被唤醒后将再次竞争相应的互斥锁，所以必须小心使用 `pthread_cond_broadcast` 函数。
 
### 释放条件变量 `pthread_cond_destroy`

```c
#include <pthread.h>
int pthread_cond_destroy(pthread_cond_t *cv);
```

返回值：函数成功返回 0；任何其他返回值都表示错误

释放条件变量。

注意：条件变量占用的空间并未被释放。
 
### 唤醒丢失问题
在线程未获得相应的互斥锁时调用 `pthread_cond_signal` 或 `pthread_cond_broadcast` 函数可能会引起唤醒丢失问题。

唤醒丢失往往会在下面的情况下发生：

* 一个线程调用 `pthread_cond_signal` 或 `pthread_cond_broadcast` 函数；
* 另一个线程正处在测试条件变量和调用 `pthread_cond_wait` 函数之间；
* 没有线程正在处在阻塞等待的状态下。

### 例子

#### 示例 1

```c
#include <stdio.h>
#include <pthread.h>

#define TCOUNT 10
#define WATCH_COUNT 12

int count = 0;
pthread_mutex_t count_mutex = PTHREAD_MUTEX_INITIALIZER;
pthread_cond_t count_threshold_cv = PTHREAD_COND_INITIALIZER;
int  thread_ids[3] = {0,1,2};
void* inc_count(void*);
void* watch_count(void*);

extern int
main(void)
{
    int      i;
    pthread_t threads[3];
    pthread_create(&threads[0],NULL,inc_count, &thread_ids[0]);
    pthread_create(&threads[1],NULL,inc_count, &thread_ids[1]);
    pthread_create(&threads[2],NULL,watch_count, &thread_ids[2]);
    for (i = 0; i < 3; i++) {
        pthread_join(threads[i], NULL);
    }
    return 0;
}

void* watch_count(void *idp) {
    pthread_mutex_lock(&count_mutex);
    while (count < WATCH_COUNT) {
        pthread_cond_wait(&count_threshold_cv,
                &count_mutex);
        printf("watch_count(): Thread %d,Count is %d\n",
                *(int*)idp, count);
    }
    pthread_mutex_unlock(&count_mutex);
    return 0;
}

void* inc_count(void *idp) {
    int i;
    for (i =0; i < TCOUNT; i++) {
        pthread_mutex_lock(&count_mutex);
        count++;
        printf("inc_count(): Thread %d, old count %d, new count %d\n",
                *(int*)idp, count - 1, count );
        if (count == WATCH_COUNT)
            pthread_cond_signal(&count_threshold_cv);
        pthread_mutex_unlock(&count_mutex);
    }
    return 0;
}
```

#####  运行结果

```
➜  pthread  gcc -Wall -pthread -o pthread_cond1 pthread_cond1.c
➜  pthread  ./pthread_cond1                                    
inc_count(): Thread 1, old count 0, new count 1
inc_count(): Thread 1, old count 1, new count 2
inc_count(): Thread 1, old count 2, new count 3
inc_count(): Thread 1, old count 3, new count 4
inc_count(): Thread 1, old count 4, new count 5
inc_count(): Thread 1, old count 5, new count 6
inc_count(): Thread 1, old count 6, new count 7
inc_count(): Thread 1, old count 7, new count 8
inc_count(): Thread 1, old count 8, new count 9
inc_count(): Thread 1, old count 9, new count 10
inc_count(): Thread 0, old count 10, new count 11
inc_count(): Thread 0, old count 11, new count 12
watch_count(): Thread 2,Count is 12
inc_count(): Thread 0, old count 12, new count 13
inc_count(): Thread 0, old count 13, new count 14
inc_count(): Thread 0, old count 14, new count 15
inc_count(): Thread 0, old count 15, new count 16
inc_count(): Thread 0, old count 16, new count 17
inc_count(): Thread 0, old count 17, new count 18
inc_count(): Thread 0, old count 18, new count 19
inc_count(): Thread 0, old count 19, new count 20
```
##### 示例 2

```c
/* 这个例子是为了保证 COUNT_HALT1 到 COUNT_HALT2 的加法由 functionCount2 来完成，但我加了两个 sleep 就全变了...*/
#include <stdio.h>
#include <stdlib.h>
#include <pthread.h>
#include <unistd.h>

pthread_mutex_t count_mutex     = PTHREAD_MUTEX_INITIALIZER;
pthread_mutex_t condition_mutex = PTHREAD_MUTEX_INITIALIZER;
pthread_cond_t  condition_cond  = PTHREAD_COND_INITIALIZER;

void *functionCount1();
void *functionCount2();
int  count = 0;
#define COUNT_DONE  10
#define COUNT_HALT1  3
#define COUNT_HALT2  6

int main()
{
   pthread_t thread1, thread2;

   pthread_create( &thread1, NULL, &functionCount1, NULL);
   pthread_create( &thread2, NULL, &functionCount2, NULL);
   pthread_join( thread1, NULL);
   pthread_join( thread2, NULL);

   exit(0);
}

void *functionCount1()
{
   for(;;)
   {
       pthread_mutex_lock( &condition_mutex );
       while( count >= COUNT_HALT1 && count <= COUNT_HALT2 )
       {
           pthread_cond_wait( &condition_cond, &condition_mutex );
       }
       pthread_mutex_unlock( &condition_mutex );

       //sleep(1);

       pthread_mutex_lock( &count_mutex );
       count++;
       printf("Counter value functionCount1: %d\n",count);
       pthread_mutex_unlock( &count_mutex );

       if(count >= COUNT_DONE) return(NULL);
   }
}

void *functionCount2()
{
    for(;;)
    {
       pthread_mutex_lock( &condition_mutex );
       if( count < COUNT_HALT1 || count > COUNT_HALT2 )
       {
          pthread_cond_signal( &condition_cond );
       }
       pthread_mutex_unlock( &condition_mutex );

       //sleep(1);

       pthread_mutex_lock( &count_mutex );
       count++;
       printf("Counter value functionCount2: %d\n",count);
       pthread_mutex_unlock( &count_mutex );

       if(count >= COUNT_DONE) return(NULL);
    }

}
```

##### 运行结果

```
➜  pthread  gcc -Wall -pthread -o pthread_cond2 pthread_cond2.c
➜  pthread  ./pthread_cond2                                  
Counter value functionCount2: 1
Counter value functionCount2: 2
Counter value functionCount2: 3
Counter value functionCount2: 4
Counter value functionCount2: 5
Counter value functionCount2: 6
Counter value functionCount2: 7
Counter value functionCount2: 8
Counter value functionCount2: 9
Counter value functionCount2: 10
Counter value functionCount1: 11
```

整理来自：[Reference1](http://blog.csdn.net/icechenbing/article/details/7662026)
          [Reference2](http://maxim.int.ru/bookshelf/PthreadsProgram/htm/r_28.html)
          [Reference3](http://www.cs.cmu.edu/afs/cs/academic/class/15492-f07/www/pthreads.html#SYNCHRONIZATION)

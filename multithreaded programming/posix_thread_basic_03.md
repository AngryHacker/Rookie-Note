注：本文涉及到的 glibc 版本为 2.11，若无特别说明，.表示 glibc-2.11 源代码目录，本文为 /usr/src/glibc-2.11。

### 基本概念
临界区：一个存取共享资源的代码段，而这些共享资源无法同时被多个线程访问；即影响共享数据的代码段。
 
线程同步方法
* 确保对相同/相关数据的内存访问互斥地进行，即一次只能允许一个线程写数据，其他线程必须等待
* Pthreads 使用特殊形式的 Edsger Dijkstra 信号灯——互斥量
* mutex: mutual(相互)，exclusion(排斥)

### 互斥量的例子
下图显示了共享互斥量的三个线程的时序图。

![lock](https://github.com/AngryHacker/ocean/blob/master/creative/image/thread_lock.png)

#### 说明
* 处于标圆形框之上的线段表示相关的线程没有拥有互斥量
* 处于圆形框中心线之上的线段表示相关的线程等待互斥量
* 处于圆形框中心线之下的线段表示相关的线程拥有互斥量

#### 过程描述

* 最初，互斥量没有被加锁
* 当线程 1 试图加锁该互斥量时，因为没有竞争，线程1立即加锁成功，对应线段也移到中心线之下
* 然后线程 2 试图加锁互斥量，由于互斥量已经被加锁，所以线程2被阻塞，对应线段在中心线之上
* 接着，线程 1 解锁互斥量，于是线程 2 解除阻塞，并对互斥量加锁成功
* 然后，线程 3 试图加锁互斥量，同样被阻塞
* 此时，线程 1 调用函数 pthread_mutext_trylock 试图加锁互斥量，而立即返回 EBUSY
* 然后，线程 2 解锁互斥量，解除线程3的阻塞，线程 3 加锁成功
* 最后，线程 3 完成工作，解锁互斥量

### 互斥量定义
#### 64位系统
file: /usr/include/bits/pthreadtypes.h

```C++
/* Data structures for mutex handling.  The structure of the attribute
   type is not exposed on purpose.  */
typedef union
{
  struct __pthread_mutex_s
  {
    int __lock;
    unsigned int __count;
    int __owner;
#if __WORDSIZE == 64
    unsigned int __nusers;
#endif
    /* KIND must stay at this position in the structure to maintain
       binary compatibility.  */
    int __kind;
#if __WORDSIZE == 64
    int __spins;
    __pthread_list_t __list;
# define __PTHREAD_MUTEX_HAVE_PREV      1
#else
    unsigned int __nusers;
    __extension__ union
    {
      int __spins;
      __pthread_slist_t __list;
    };
#endif
  } __data;
  char __size[__SIZEOF_PTHREAD_MUTEX_T];
  long int __align;
} pthread_mutex_t;

typedef union
{
  char __size[__SIZEOF_PTHREAD_MUTEXATTR_T];
  long int __align;
} pthread_mutexattr_t;
```

该定义来 自glibc，其在 glibc 代码中的位置为./nptl/sysdeps/unix/sysv/linux/x86_64/bits/pthreadtypes.h。64 位系统在安 装glibc 的时候会自动拷贝该文件(x86_64版本)到/usr/include/bits目录。

 
其中:

```C++
# define __SIZEOF_PTHREAD_MUTEX_T 40
# define __SIZEOF_PTHREAD_MUTEXATTR_T 4
```

关于 __pthread_list_t(双向链表)和 __pthread_slist_t(单向链表)的定义可参考源代码。

#### 32位系统
file: /usr/include/bits/pthreadtypes.h

```C++
/* Data structures for mutex handling.  The structure of the attribute
   type is not exposed on purpose.  */
typedef union
{
  struct __pthread_mutex_s
  {
    int __lock;
    unsigned int __count;
    int __owner;
    /* KIND must stay at this position in the structure to maintain
       binary compatibility.  */
    int __kind;
    unsigned int __nusers;
    __extension__ union
    {
      int __spins;
      __pthread_slist_t __list;
    };
  } __data;
  char __size[__SIZEOF_PTHREAD_MUTEX_T];
  long int __align;
} pthread_mutex_t;

typedef union
{
  char __size[__SIZEOF_PTHREAD_MUTEXATTR_T];
  long int __align;
} pthread_mutexattr_t;
```

该定义来自 glibc，其 在glibc 代码中的位置为./nptl/sysdeps/unix/sysv/linux/i386/bits/pthreadtypes.h。32 位系统在安装 glibc 的时候会自动拷贝该文件(i386版本)到 /usr/include/bits 目录。

其中：

```C++
#define __SIZEOF_PTHREAD_MUTEX_T 24
#define __SIZEOF_PTHREAD_MUTEXATTR_T 4
```

#### pthread_mutex_t 结构的内容
如下是在 64 位系统的实验结果。

```shell
(gdb) p data.mutex 
$1 = {
  __data = {
    __lock = 0, 
    __count = 0, 
    __owner = 0, 
    __nusers = 0, 
    __kind = 0, 
    __spins = 0, 
    __list = {
      __prev = 0x0, 
      __next = 0x0
    }
  }, 
  __size = '\000' <repeats 39 times>, 
  __align = 0
}
```

### 互斥量初始化与销毁
#### 初始化
互斥量使用原则：使用前必须初始化，而且只被初始化一次。

##### 静态初始化
使用宏 PTHREAD_MUTEX_INITIALIZER 声明具有默认属性的静态互斥量：
file: /usr/include/pthread.h

```c++
/* Mutex initializers.  */
#if __WORDSIZE == 64
# define PTHREAD_MUTEX_INITIALIZER \
  { { 0, 0, 0, 0, 0, 0, { 0, 0 } } }
# ifdef __USE_GNU
#  define PTHREAD_RECURSIVE_MUTEX_INITIALIZER_NP \
  { { 0, 0, 0, 0, PTHREAD_MUTEX_RECURSIVE_NP, 0, { 0, 0 } } }
#  define PTHREAD_ERRORCHECK_MUTEX_INITIALIZER_NP \
  { { 0, 0, 0, 0, PTHREAD_MUTEX_ERRORCHECK_NP, 0, { 0, 0 } } }
#  define PTHREAD_ADAPTIVE_MUTEX_INITIALIZER_NP \
  { { 0, 0, 0, 0, PTHREAD_MUTEX_ADAPTIVE_NP, 0, { 0, 0 } } }
# endif
#else
# define PTHREAD_MUTEX_INITIALIZER \
  { { 0, 0, 0, 0, 0, { 0 } } }
# ifdef __USE_GNU
#  define PTHREAD_RECURSIVE_MUTEX_INITIALIZER_NP \
  { { 0, 0, 0, PTHREAD_MUTEX_RECURSIVE_NP, 0, { 0 } } }
#  define PTHREAD_ERRORCHECK_MUTEX_INITIALIZER_NP \
  { { 0, 0, 0, PTHREAD_MUTEX_ERRORCHECK_NP, 0, { 0 } } }
#  define PTHREAD_ADAPTIVE_MUTEX_INITIALIZER_NP \
  { { 0, 0, 0, PTHREAD_MUTEX_ADAPTIVE_NP, 0, { 0 } } }
# endif
#endif
```

该文件在 glibc 代码中的位置为 ./nptl/sysdeps/pthread/pthread.h。

##### 动态初始化
* 通过 `pthread_mutex_init()` 调用动态初始化互斥量
* 使用场合
  * 当使用 malloc 动态分配一个包含互斥量的数据结构时，应使用动态初始化
  * 若要初始化一个非缺省属性的互斥量，必须使用动态初始化
  * 也可动态初始化静态声明的互斥量，但必须保证每个互斥量在使用前被初始化，而且只能被初始化一次
  
动态初始化代码可参考./nptl/pthread_mutex_init.c 文件。其中 `__pthread_mutex_init()` 函数即对mutex的各个feild进行初始化。

#### 销毁互斥量
使用 `pthread_mutex_destroy()` 释放互斥量。
 
注意
* 当确信没有线程在互斥量上阻塞，且互斥量没有被锁住时，可以立即释放
* 不需要销毁一个使用 PTHREAD_MUTEX_INITIALIZER 宏静态初始化的互斥量
 
销毁互斥量代码可参考 ./nptl/pthread_mutex_destroy.c 文件。其中` __pthread_mutex_destroy()` 函数设置 mutex 的相应字段使其不可用。代码如下:

```c++
int
__pthread_mutex_destroy (mutex)
     pthread_mutex_t *mutex;
{
  if ((mutex->__data.__kind & PTHREAD_MUTEX_ROBUST_NORMAL_NP) == 0
      && mutex->__data.__nusers != 0)
    return EBUSY;

  /* Set to an invalid value.  */
  mutex->__data.__kind = -1;

  return 0;
}
```

整理来自：[Reference](http://blog.csdn.net/livelylittlefish/article/details/8096595)

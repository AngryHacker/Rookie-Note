### 简介
POSIX thread 简称为 pthread，Posix 线程是一个 POSIX 标准线程。该标准定义内部 API 创建和操纵线程。

### 作用
线程库实行了 POSIX 线程标准通常称为 pthreads. pthreads 是最常用的 POSIX 系统如 Linux 和 Unix，而微软 Windowsimplementations 同时存在。举例来说，pthreads-w32 可支持 MIDP 的 pthread。

Pthreads 定义了一套 C 程序语言类型、函数与常量，它以 pthread.h 头文件和一个线程库实现。

### 数据结构与函数
#### 数据结构

<table>
  <tr>
    <td>pthread_t</td>
    <td>线程句柄</td>
  </tr>
  <tr>
    <td>pthread_attr_t</td>
    <td>线程属性</td>
  </tr>
</table>

#### 线程操纵函数（省略参数）

<table>
  <tr>
    <td>pthread_create</td>
    <td>创建一个线程</td>
  </tr>
  <tr>
    <td>pthread_exit</td>
    <td>终止当前线程 </td>
  </tr>
  <tr>
    <td>pthread_cancel</td>
    <td>中断另外一个线程的运行</td>
  </tr>
  <tr>
    <td>pthread_join</td>
    <td>阻塞当前的线程，直到另外一个线程运行结束</td>
  </tr>
  <tr>
    <td>pthread_attr_init</td>
    <td>初始化线程的属性</td>
  </tr>
  <tr>
    <td>pthread_attr_setdetachstate</td>
    <td>设置脱离状态的属性（决定这个线程在终止时是否可以被结合）</td>
  </tr>
  <tr>
    <td>pthread_attr_getdetachstate</td>
    <td>获取脱离状态的属性</td>
  </tr>
  <tr>
    <td>pthread_attr_destroy</td>
    <td>删除线程的属性</td>
  </tr>
  <tr>
    <td>pthread_kill</td>
    <td>向线程发送一个终止信号</td>
  </tr>
</table>

#### 同步函数
用于 mutex 和条件变量

<table>
  <tr>
    <td>pthread_mutex_init</td>
    <td>初始化互斥锁</td>
  </tr>
  <tr>
    <td>pthread_mutex_destroy</td>
    <td>删除互斥锁</td>
  </tr>
  <tr>
    <td>pthread_mutex_lock</td>
    <td>占有互斥锁（阻塞操作）</td>
  </tr>
  <tr>
    <td>pthread_mutex_trylock</td>
    <td>试图占有互斥锁（不阻塞操作）。当互斥锁空闲时将占有该锁；否则立即返回　</td>
  </tr>
  <tr>
    <td>pthread_mutex_unlock</td>
    <td>释放互斥锁</td>
  </tr>
  <tr>
    <td>pthread_cond_init</td>
    <td>初始化条件变量</td>
  </tr>
  <tr>
    <td>pthread_cond_destroy</td>
    <td>销毁条件变量</td>
  </tr>
  <tr>
    <td>pthread_cond_wait</td>
    <td>等待条件变量的特殊条件发生</td>
  </tr>
  <tr>
    <td>pthread_cond_signal</td>
    <td>唤醒第一个调用pthread_cond_wait()而进入睡眠的线程</td>
  </tr>
</table>

#### Thread-local storage（线程特有数据） 
<table>
  <tr>
    <td>pthread_key_create</td>
    <td>分配用于标识进程中线程特定数据的键</td>
  </tr>
  <tr>
    <td>pthread_setspecific</td>
    <td>为指定线程特定数据键设置线程特定绑定 </td>
  </tr>
  <tr>
    <td>pthread_getspecific</td>
    <td>获取调用线程的键绑定，并将该绑定存储在 value 指向的位置中</td>
  </tr>
  <tr>
    <td>pthread_key_delete</td>
    <td>销毁现有线程特定数据键</td>
  </tr>
</table>

#### 工具函数
<table>
  <tr>
    <td>pthread_equal</td>
    <td>对两个线程的线程标识号进行比较</td>
  </tr>
  <tr>
    <td>pthread_detac</td>
    <td>分离线程</td>
  </tr>
  <tr>
    <td>pthread_self</td>
    <td>查询线程自身线程标识号</td>
  </tr>
</table>

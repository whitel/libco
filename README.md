# libco简介

- libco 是腾讯开源的一个有趣的协程基础库，仅有的几个函数接口 co_create/co_resume/co_yield ，再配合 co_poll，可以支持同步编码方式实现“异步”性能。
- 代码的实现十分清晰明了，在阅读源码时，建议大家先理清楚各个数据结构，再深入探究。

## 个人理解

1. 所有的协程都在一个线程环境中（所有协程共用该env，它就是一个全局变量），因此对临界资源访问，不需要加锁
2. 一个线程环境在某一时刻，只有一个协程在执行，当该协程卡住时，后面的协程任务也将会卡住
3. 相较于线程/进程，协程控制块更小，约128K，在用户态实现切换（不经过内核态），所以，性能极高
4. 适用场景：高并发IO密集型

## 数据结构与机制

1. dlsym实现的HOOK机制（打桩）：对原系统调用进行改造
2. epoll实现事件监听（读写事件/超时事件）：超时事件采用“时间轮”算法
3. 协程上下文切换采用汇编方式
4. 实现了自己的条件变量：cond实体本质上是一个双向链表

---

---

---



下面才开始正式介绍我认知的libco… … :smiley:



# 协程

- 协程是一种轻量级线程，可以在用户态实现上下文的切换，不像线程那样，上下文切换由OS操控，要涉及用户态和内核态的切换。而，协程的切换完全是在用户态，切换速度快，开销小。
- 所有的协程都处在一个线程中，约定：在同一个时刻，该线程只能有一个协程处于running状态，其他线程都会阻塞。正因为如此，协程的同步不像线程那样需要加锁。
- 协程在应用层实现上下文切换，过程：
  - ① 将当前协程的上下文保存在寄存器中
  - ② 切走当前协程，执行其他的协程
  - ③ 执行完其他协程后，再切回之前协程的上下文，继续执行



---



## libco协程实现原理

### :slightly_smiling_face: 全局唯一的协程环境

- :small_airplane: pCallSTack 协程调用栈（libco的协程是非对称协程）
  - 有点类似于函数调用栈，假如在协程A中开启了协程B，协程B中开启了协程C，那么pCallStack=[A,B,C]，栈顶元素始终是当前正在running的协程
  - co_create
    - 创建协程，为协程绑定上下文（堆栈、协程函数）
  - co_resume
    - 将co保存到pCallStack中
    - co_swap(curr, co) 切换到目标协程co，运行co绑定的协程函数
- :small_airplane: epoll管理者: 都采用epoll的方式，对(IO事件/超时事件)2种事件进行监管
  - IO事件：epoll的fd事件监听
  - 超时事件（时间轮）：epoll的超时参数

---

### :slightly_smiling_face: poll  （采用hook机制，对原函数poll进行改造）

```c++
poll(NULL, 0, 1000);    // 超时事件
poll( &pf,1,timeout );  // 读写事件
```

- 将(IO事件/超时事件)托管给epoll中监管，如何实现托管的呢？
  - IO事件
    - co_epoll_ctl: 将其添加到epoll中，监控其上的IO事件的到来
    - co_yield_env: 从当前事件的协程，切换到co_event_loop协程
    - 当读写事件触发后，co_event_loop会将该事件移到就绪链表中
    - 之后，处理就绪链表中就绪的事件，即：调用注册的事件协程的回调函数（切回该事件的协程），实际上切回的位置是co_yield_env。之后，继续读写操作
  - 超时事件
    - 超时事件，添加到小根堆中，由epoll监管
    - co_yield_env: 从当前事件的协程，切换到co_event_loop协程
    - 当超时事件触发后，co_event_loop会将该事件移到就绪链表中
    - 之后，处理就绪链表中就绪的事件，即：调用注册的事件协程的回调函数（切回该事件的协程），切回来后，继续执行该协程下面的操作

最后，再给出关于poll函数的详细解释

```c++
 /*
 *   poll函数在hook目录下
 *      注意: 在调用read\write\recv\send\connect等函数时, timeout_ms永远不为0
 *
 * @detail
 *         poll函数的核心:
 *              将(IO事件/超时事件)托管给epoll管理
 *              切走: 切到epoll主协程, 由epoll监控事件的就绪
 *              切回: 当事件就绪后，执行回调函数, 再重新切回来，继续处理该协程事件
 *
 * @brief  该函数用来添加一个read\write等事件
 *            添加完事件后, 将会切走协程
 *            当事件触发后, 将会切回来
 * @note
 *    如果你真的不懂, 也不耽误你的使用, poll的使用场景无非就分为2类:
 *        case1: 直接调用poll, 用于超时事件, 如: poll(NULL, 0, 1000);
 *        case2: read/write函数中调用了poll
 *    牢牢记住, 
 *       poll函数的使用一定要在co_create创建的协程函数中, 它内部由libco给我们提供了协程的切换
 *       你可以把它当作是同步调用来使用(本质上, libco将它采用hook机制, 封装成 “协程+epoll异步调用”的方式了 )
 */
```

----

写在最前，上面已经对 `poll` 函数进行了详细剖析，再来看下面的系统调用，就会很简单

### :slightly_smiling_face: read/write/connect/…

- 都采用了HOOK机制
- 在函数内部，都调用了上面讲到的poll

以read函数调用举例

- 传统的read函数，要进行2步：
  - 阻塞等待数据的到来 （`数据没来，就一直卡在这里等待，别人无法继续执行，这显然是不行的哇`）
  - 数据到来后，阻塞的读取数据 （`libco，该步骤仍然是阻塞调用，没有改变该步`）
- 本文中的read，采用`epoll + 协程切换`，将`同步阻塞`调用转变为`异步调用`
  - 当调用read函数后，不会阻塞等待数据的到来
    - 新建协程任务，co_create，协程回调函数中执行了read调用（该read函数是hook的）
    - hook的read函数：
      - ① 将read事件添加到epoll中管理
      - ② co_yield_env：（协程切换）read事件所处的协程  --> co_event_loop主协程
    - co_event_loop主协程，通过epoll监听协程任务事件的到来；当数据已经准备好时，epoll_wait将会捕获到该读事件，此时会调用该读事件注册的回调函数（回调函数的功能：co_resume(co) 切回该读事件协程，实际上切回的位置仍然是②中的co_yield_env处）
  - 经过以上的步骤，数据已经到来了，之后就可以阻塞的读取数据了…

----

经过上面的read介绍，可以发现：

- 对于所有的（IO事件/超时事件），都应该在外部包裹一个协程，代码的书写就显而易见了。

举例：下面代码书写方式肯定不对，此处只是为了描述原理

```c++
static void *readwrite_routine( void *arg ){
    while (1) {
        read(fd, xx, xx);  // read内部调用了poll
        				   //     将read事件托管给epoll监听，切回到主协程
        				   // 当读事件到来后，会切回来，执行读取数据的操作，此时read函数才真正的执行完毕
    }
}

int main(){
    stCoRoutine_t* co;
	
    co_create(&co, NULL,readwrite_routine, xx);  // 1. 首先要为read事件创建一个协程
    co_resume( co );   // 切换到读协程中去
    
    co_eventloop( co_get_epoll_ct(),0,0 );  // 监听事件的到来
    						// 当读事件到来后，会再次切回read函数的位置
}
```



---

epoll实现的反应堆模式（与libevent中的epoll本质上没有任何差异），该模式非常常见，此处就不在赘述… 

### :slightly_smiling_face: co_eventloop 事件循环

- 事件循环一直处于 `主协程` 中
- read/write等IO事件、超时事件，都由它来监管
- 它的实现很简单，就是某个事件需要我来监管时，你就注册事件就绪的回调函数，然后调用co_epoll_ctl将事件安插到epoll中，之后，我调用epoll_wait获取就绪事件，然后，回调你注册的回调函数


### :slightly_smiling_face: 条件变量

libco的条件变量和其他的函数不太一样，它并不是简单的hook一下，而是根据libco的架构重新设计了一个协程版的条件变量。（条件变量的实现，与超时事件的实现类似）

:small_orange_diamond:co_cond_timedwait

- 在协程co中，调用 `co_cond_timedwait` 后，因为wait函数内部调用了co_yield_ct，所以，该将会使得协程co挂起，切回到主协程co_eventloop。（co_cond_timedwait的内部实现细节）
  - 将协程事件挂到该条件变量链表中

:small_orange_diamond:到时、co_cond_signal / co_cond_broadcast

- 条件变量协程被唤醒的情况分为2中：① 到时 ② 被co_cond_signal/co_cond_broadcast唤醒

  ① 到时

  - 与超时事件一样，由epoll监控超时事件的到来，将超时事件添加到就绪列表

  ② 被co_cond_signal/co_cond_broadcast唤醒

  - 当执行co_cond_signal/co_cond_broadcast后，将会等待在条件变量上的事件添加到就绪列表

- 说明：被加入到就绪列表中的事件，将会被触发

```c++
/* 将当前协程, 在条件变量link上, 等待 (超时事件设为ms毫秒)
 *
 *  @param  [in]  link  条件变量, 外部定义env->cond   
 *  @param  [in]  ms    等待超时时间
 *                ms <= 0 永久等待
 *                ms >  0 等待毫秒数 
 */
int co_cond_timedwait( stCoCond_t *link, int ms )
```

```c++
/* co_cond_signal仅仅是将等待在条件变量上的某一个事件，移动到激活链表中
 *   @detail
 *      由于co_eventloop中的co_epoll_wait函数的超时事件是1ms,
 *      所以, 加入到激活队列中的事件最多会在1ms后触发
 */
int co_cond_signal( stCoCond_t *si )
```

```c++
// 仅仅是将等待在条件变量上的所有事件，移动到激活链表中
int co_cond_broadcast( stCoCond_t *si )
```


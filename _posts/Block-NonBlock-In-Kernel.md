---
published: true
date: {}
tags:
  - Linux
  - Netfilter
  - Kernel
  - Blocking
  - Thread
title: kernel中的阻塞与非阻塞
---

“要效率，就用NON-BLOCK操作吧！”， 这大概是很多program，特别是玩网络的深信不移的话。那么kernel里是如何实现阻塞和非阻塞的呢？
kernel里一般利用filp->f_flag中的O_NONBLOCK位，或者诸如release socket时利用的sock->sk->linger标志，来判断该操作是否可以阻塞。如果可以，则将进程设置成TASK_INTERRUPTIBLE 或者 TASK_UNINTERRUPTIBLE， 加入到该操作相关的等待队列，然后探测异步操作是否结束，如果操作结束，则该进程继续执行，否则schedule，阻塞自己。当阻塞的进程由于异步事件的到来而被唤醒时，再次判断操作是否完成，如果还未完成，继续schedule，否则，进程重新执行下去。
这里就存在一个问题，如何在操作未完成时强制唤醒阻塞进程呢？如果进程阻塞时，进程状态设置成TASK_INTERRUPTIBLE，那么一个signal将会唤醒进程。否则只能等待操作完成。所以如果你自己再kernel里写了一些能阻塞的操作，并且希望异步的停止阻塞，那么请确保该阻塞操作是可被中断的。下面来看一个例子：
下面这个函数是被tcp_ipv4.c:tcp_accept()中调用的，用来等待一个连接到来。

``` C
/* 注意参数中的timeo，如果是非阻塞，值为0 */
static int wait_for_connect(struct sock * sk, long timeo)
{
    DECLARE_WAITQUEUE(wait, current);
    int err;
 
    add_wait_queue_exclusive(sk->sleep, &wait); // 加入到sk->sleep的等待队列上，准备阻塞。
    for (;;) {  // 循环判断操作是否完成，未完成则schedule
        current->state = TASK_INTERRUPTIBLE; // 可被signal中断
        release_sock(sk);
        if (sk->tp_pinfo.af_tcp.accept_queue == NULL) // 接收队列中是否有新连接？
            timeo = schedule_timeout(timeo); // 无新连接，则执行重新调度。如果timeo==0 schedule_timeout的行为类似schedule，但是进程确被重新放入running task队列中，等待运行，所以会被快速的执行。
        lock_sock(sk);
        err = 0;
        if (sk->tp_pinfo.af_tcp.accept_queue)
            break;
        err = -EINVAL;
        if (sk->state != TCP_LISTEN)
            break;
        err = sock_intr_errno(timeo);
        if (signal_pending(current))  // 如果有信号到来，则跳出循环，停止阻塞
            break;
        err = -EAGAIN;
        if (!timeo) // 超时，或者是非阻塞操作，则跳出循环
            break;
    }
    current->state = TASK_RUNNING;
    remove_wait_queue(sk->sleep, &wait); // 把该进程从等待队列中删除
    return err;
}
```

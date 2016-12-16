---
published: true
date: 2007-11-14T13:00:00.000Z
tags:
  - Linux
  - Timekeeping
  - Kernel
  - Clocksource
  - HRTimer
title: Linux Kernel Timekeeping
---

kernel-2.6.22中的arm arch加入了对dynticks, clocksource/event支持. imx31的BSP在clock这里有一些改动. 找了些kernel clock及timer子系统近来的变化, 总结一下。
一般来说Soft-Timer (timer wheel / hrtimer) 都是由Hardware-Timer(时钟中断之类)以及相关的clock source(e.g GPT in Soc)驱动, 所以我打算先从clock这层开始介绍, 接着是soft-timer, kernel timekeeping, 最后来看一些应用.

## Clock Source

clock source定义了一个clock device的基本属性及行为, 这些clock device一般都有计数, 定时, 产生中断能力, 比如GPT. 结构定义如下:
``` C
struct clocksource {
    char *name;
    struct list_head list;
    int rating;
    cycle_t (*read)(void);
    cycle_t mask;
    u32 mult; /* cycle -> xtime interval, maybe two clock cycle trigger one interrupt (one xtime interval) */
    u32 shift;
    unsigned long flags;
    cycle_t (*vread)(void);
    void (*resume)(void);
    /* timekeeping specific data, ignore */
    cycle_t cycle_interval; /* just the rate of GPT count per OS HZ */
    u64    xtime_interval; /* xtime_interval = cycle_interval * mult. */
    cycle_t cycle_last ____cacheline_aligned_in_smp; /* last cycle in rate count */
    u64 xtime_nsec; /* cycle count, remain from xtime.tv_nsec 
                     * now nsec rate count offset = xtime_nsec +                              
                     * xtime.tv_nsec << shift */
    s64 error;
};
```

最重要的成员是read(), cycle_last和cycle_interval. 分别定义了读取clock device count 寄存器当前计数值接口, 保存上一次周期计数值和每个tick周期间隔值. 这个结构内的值, 无论是cycle_t, 还是u64类型(实际cycle_t就是u64)都是计数值(cycle), 而不是nsec, sec和jiffies. read()是整个kernel读取精确的单调时间计数的接口, kernel会用它来计算其他时间, 比如:jiffies, xtime. 
clocksource的引入, 解决了之前kernel各个arch都有自己的clock device的管理方式, 基本都隐藏在MSL层, kernel core 及driver很难访问的问题. 它导出了以下接口:
1. clocksource_register() 注册clocksource
2. clocksource_get_next() 获取当前clocksource设备
3. clocksource_read() 读取clock, 实际跑到clocksource->read()

当driver处理的时间精度比较高的时, 可以通过上面的接口, 直接拿clock device来读.
当然目前ticker时钟中断源也会以clocksource的形式存在.

## Clock Event

Clock event的主要作用是分发clock事件及设置下一次触发条件. 在没有clock event之前, 时钟中断都是周期性地产生, 也就是熟知的jiffies和HZ. 

Clock Event device主要的结构:

``` C
struct clock_event_device {
    const char        *name;
    unsigned int        features;
    unsigned long        max_delta_ns;
    unsigned long        min_delta_ns;
    unsigned long        mult;
    int            shift;
    int            rating;
    int            irq;
    cpumask_t        cpumask;
    int            (*set_next_event)(unsigned long evt,
                         struct clock_event_device *);
    void            (*set_mode)(enum clock_event_mode mode,
                     struct clock_event_device *);
    void            (*event_handler)(struct clock_event_device *);
    void            (*broadcast)(cpumask_t mask);
    struct list_head    list;
    enum clock_event_mode    mode;
    ktime_t            next_event;
};
```

最重要的是set_next_event(), event_handler(). 前者是设置下一个clock事件的触发条件, 一般就是往clock device里重设一下定时器. 后者是event handler, 事件处理函数. 该处理函数会在时钟中断ISR里被调用. 如果这个clock用来做为ticker时钟, 那么handler的执行和之前kernel的时钟中断ISR基本相同, 类似timer_tick(). 事件处理函数可以在运行时动态替换, 这就给kernel一个改变整个时钟中断处理方式的机会, 也就给highres tick及dynamic tick一个动态挂载的机会. 目前kernel内部有periodic/highres/dynamic tick三种时钟中断处理方式. 后面会介绍.

## hrtimer & timer wheel

首先说一下timer wheel. 它就是kernel一直采用的基于jiffies的timer机制, 接口包括init_timer(), mod_timer(), del_timer()等, 很熟悉把. hrtimer 的出现, 并没有抛弃老的timer wheel机制(也不太可能抛弃:)). hrtimer做为kernel里的timer定时器, 而timer wheel则主要用来做timeout定时器. 分工比较明确. hrtimers采用红黑树来组织timers, 而timer wheel采用链表和桶. hrtimer精度由原来的timer wheel的jiffies提高到nanosecond. 主要用于向应用层提供nanosleep, posix-timers和itimer接口, 当然驱动和其他子系统也会需要high resolution的timer. kernel 里原先每秒周期性地产生HZ个ticker(中断), 被在下一个过期的hrtimer的时间点上产生中断代替. 也就是说时钟中断不再是周期性的, 而是由timer来驱动(靠clockevent的set_next_event接口设置下一个事件中断), 只要没有hrtimer加载, 就没有中断. 但是为了保证系统时间(进程时间统计, jiffies的维护)更新, 每个tick_period(NSEC_PER_SEC/HZ, 再次强调hrtimer精度是nsec)都会有一个叫做tick_sched_timer的hrtimer加载.
接下来对比一下, hrtimer引入之前及之后, kernel里时钟中断的处理的不同. (这里都是基于arm arch的source去分析)

### no hrtimer

kernel 起来, setup_arch()之后的time_init()会去初始化相应machine结构下的timer. 初始化timer函数都在各个machine的体系结构代码中, 初始化完硬件时钟, 注册中断服务函数, 使能时钟中断. 中断服务程序会清中断, 调用timer_tick(), 它执行: 
``` C
profile_tick(); /* kernel profile, 不是很了解 */
do_timer(1); /* 更新jiffies */
update_process_times(); /* 计算进程耗时, 唤起TIMER_SOFTIRQ(timer wheel), 重新计算调度时间片等等 */
```

最后中断服务程序设置定时器, 使其在下一个tick产生中断.

这样的框架, 使得high-res的timer很难加入. 所有中断处理code都在体系结构代码里被写死, 并且代码重用率很低, 毕竟大多的arch都会写同样的中断处理函数.

### hrtimer

kernel 里有了clockevent/source的引入, 就把clocksource的中断以一种事件的方式被抽象出来. 事件本身的处理交给event handler. handler可以在kernel里做替换从而改变时钟中断的行为. 时钟中断ISR会看上去象这样:

``` C
static irqreturn_t timer_interrupt(int irq, void *dev_id)
{
    /* clear timer interrupt flag */
    .....
    /* call clock event handler */
    arch_clockevent.event_handler(&arch_clockevent);
    ....
    return IRQ_HANDLED;
}
```

event_handler 在注册clockevent device时, 会被默认设置成tick_handle_periodic(). 所以kernel刚起来的时候, 时钟处理机制仍然是periodic的, ticker中断周期性的产生. tick_handle_periodic()会做和timer_tick差不多的事情, 然后调clockevents_program_event() => arch_clockevent.set_next_event()去设置下一个周期的定时器. tick-common.c里把原来kernel时钟的处理方式在clockevent框架下实现了, 这就是periodic tick的时钟机制.

hres tick机制在第一个TIMER SOFTIRQ里会替换掉periodic tick, 当然要符合一定条件, 比如command line里没有把hres(highres=off)禁止掉, clocksource/event支持hres和oneshot的能力. 这里的切换做的比较ugly, 作者的comments也提到了, 每次timer softirq被调度, 都要调用hrtimer_run_queues()检查一遍hres是否active, 如果能在timer_init()里就把clocksource/event的条件check过, 直接切换到hres就最好了, 不知道是不是有什么限制条件. TIMER SOFTIRQ代码如下:

``` C
static void run_timer_softirq(struct softirq_action *h)
{
    tvec_base_t *base = __get_cpu_var(tvec_bases);
    hrtimer_run_queues(); /* 有机会就切换到hres或者nohz */

    if (time_after_eq(jiffies, base->timer_jiffies))
        __run_timers(base); /* timer wheel */
}
```

切换的过程比较简单, 用hrtimer_interrupt()替换当前clockevent hander, 加载一个hrtimer: tick_sched_timer在下一个tick_period过期, retrigger下一次事件. hrtimer_interrupt ()将过期的hrtimers从红黑树上摘下来, 放到相应clock_base->cpu_base->cb_pending列表里, 这些过期timers会在HRTIMER_SOFTIRQ里执行. 然后根据剩余的最早过期的timer来retrigger下一个event, 再调度HRTIMER_SOFTIRQ. hrtimer softirq执行那些再cb_pending上的过期定时器函数. tick_sched_timer这个hrtimer在每个tick_period都会过期, 执行过程和timer_tick()差不多, 只是在最后调用hrtimer_forward将自己加载到下一个周期里去, 保证每个tick_period都能正确更新kernel内部时间统计.

## Timekeeping

Timekeeping子系统负责更新xtime, 调整误差, 及提供get/settimeofday接口. 为了便于理解, 首先介绍一些概念:

**Times in Kernel**

kernel的time基本类型:
1) system time - A monotonically increasing value that represents the amount of time the system has been running. 单调增长的系统运行时间, 可以通过time source, xtime及wall_to_monotonic计算出来.
2) wall time - A value representing the the human time of day, as seen on a wrist-watch. Realtime时间: xtime.
3) time source - A representation of a free running counter running at a known frequency, usually in hardware, e.g GPT. 可以通过clocksource->read()得到counter值
4) tick - A periodic interrupt generated by a hardware-timer, typically with a fixed interval
defined by HZ: jiffies

这些time之间互相关联, 互相可以转换.
``` C
system_time = xtime + cyc2ns(clock->read() – clock->cycle_last) + wall_to_monotonic;
real_time = xtime + cyc2ns(clock->read() – clock->cycle_last)
```
也就是说real time是从1970年开始到现在的nanosecond, 而system time是系统启动到现在的nanosecond.
这两个是最重要的时间, 由此hrtimer可以基于这两个time来设置过期时间. 所以引入两个clock base.

**Clock Base**

CLOCK_REALTIME: base在实际的wall time
CLOCK_MONOTONIC: base在系统运行system time
hrtimer可以选择其中之一, 来设置expire time, 可以是实际的时间, 也可以是相对系统的时间.
他们提供get_time()接口:
CLOCK_REALTIME 调用ktime_get_real()来获得真实时间, 该函数用上面提到的等式计算出realtime.
CLOCK_MONOTONIC 调用ktime_get(), 用system_time的等式获得monotonic time.

timekeeping提供两个接口do_gettimeofday()/do_settimeofday(), 都是针对realtime操作. 用户空间对gettimeofday的syscall也会最终跑到这里来.do_gettimeofday()会调用\_\_get_realtime_clock_ts()获得时间, 然后转成timeval.
do_settimeofday(), 将用户设置的时间更新到xtime, 重新计算xtime到monotonic的转换值, 最后通知hrtimers子系统时间变更.

``` C
int do_settimeofday(struct timespec *tv)
{
    unsigned long flags;
    time_t wtm_sec, sec = tv->tv_sec;
    long wtm_nsec, nsec = tv->tv_nsec;
    if ((unsigned long)tv->tv_nsec >= NSEC_PER_SEC)
        return -EINVAL;

    write_seqlock_irqsave(&xtime_lock, flags);

    nsec -= __get_nsec_offset();

    wtm_sec = wall_to_monotonic.tv_sec + (xtime.tv_sec - sec);
    wtm_nsec = wall_to_monotonic.tv_nsec + (xtime.tv_nsec - nsec);
    set_normalized_timespec(&xtime, sec, nsec); /* 重新计算xtime: 用户设置的时间减去上一个周期到现在的nsec */
    set_normalized_timespec(&wall_to_monotonic, wtm_sec, wtm_nsec); /* 重新调整wall_to_monotonic */
    clock->error = 0;
    ntp_clear();
    update_vsyscall(&xtime, clock);
    write_sequnlock_irqrestore(&xtime_lock, flags);
    /* signal hrtimers about time change */
    clock_was_set();

    return 0;
}
```

## Userspace Application

hrtimer的引入, 对用户最有用的接口如下:

### Clock API

获取对应clock的时间
``` C
clock_gettime(clockid_t, struct timespec *)
```

设置对应clock时间
``` C
clock_settime(clockid_t, const struct timespec *)
```

进程nano sleep
``` C
clock_nanosleep(clockid_t, int, const struct timespec *, struct timespec *)
```

获取时间精度, 一般是nanosec
``` C
clock_getres(clockid_t, struct timespec *)
```

clockid_t 定义了四种clock:

CLOCK_REALTIME - System-wide realtime clock. Setting this clock requires appropriate privileges.
CLOCK_MONOTONIC - Clock that cannot be set and represents monotonic time since some unspecified starting point.
CLOCK_PROCESS_CPUTIME_ID - High-resolution per-process timer from the CPU.
CLOCK_THREAD_CPUTIME_ID - Thread-specific CPU-time clock.
前两者前面提到了, 后两个是和进程/线程统计时间有关系, 还没有仔细研究过, 是utime/stime之类的时间. 应用层可以利用这四种clock, 提高灵活性及精度.

### Timer API

Timer 可以建立进程定时器，单次或者周期性定时。

创建定时器
``` C
int timer_create(clockid_t clockid, struct sigevent *restrict evp, timer_t *restrict timerid);
```
clockid - 在哪个clock base下创建定时器。
evp (sigevent) - 指定定时器到期后内核发送哪个信号给进程，以及信号所带参数；默认为SIGALRM。
timerid - 所建timer的id号。
在signal 处理函数里，可以通过siginfo_t.si_timerid 获得当前的信号是由哪个timer过期触发的。试验了一下，最多可创建的timer数目和ulimit里的pending signals的有关系，不能超过pending signals的数量。

获得timer的下次过期的时间。
``` C
int timer_gettime(timer_t timerid, struct itimerspec *value);  
```

设置定时器的过期时间及间隔周期。
``` C
int timer_settime(timer_t timerid, int flags, const struct itimerspec *restrict value, struct itimerspec *restrict ovalue);
```

删除定时器
``` C
int timer_delete(timer_t timerid);
```

这些系统调用都会建立一个posix_timer的hrtimer，在过期的时候发送信号给进程。

## 总结

hrtimer 及clockevent/source的引入对于kernel的实时性的提高有很大贡献，也将clock的处理从体系结构的代码中抽象了出来，增强了代码的可重用性。并且对于posix的time／timer标准有了强有力的支持，提高了用户空间的应用程序的时间处理精度及灵活性。如果应用层在使用这些 syscall时有任何不解之处，直接看看hrtimer的code，对于处理问题，理解OS的行为都有很大帮助。

参考资料:
[1] http://tglx.de/projects/hrtimers/ols2006-hrtimers.pdf
[2] http://www.linuxsymposium.org/2006/linuxsymposium_procv1.pdf
[3] Documentation/hrtimers/highres.txt
[4] Documentation/hrtimers/hrtimers.txt
[5] http://sourceforge.net/projects/high-res-timers/

---
published: false
---

最近QA报来的一个bug，引发了我对2.6.x linux线程实现的兴趣。以前一直以为pthread只是简单的利用clone() syscall来实现，2.4和2.6的kernel在这方面应该没有什么大改进。可稍微研究一下，才发现自己错了。
Linux 2.6.x kernel 加入了对POSIX标准线程的支持。而在应用层，有LinuxThread和NPTL两个不同的库，不过接口都一样，库名也都是pthread。LinuxThread是比较早的pthread实现，在glibc 2.2.x里都有集成；而NPTL(Native POSIX Thread Library)则是Linux 线程的一个新实现，它克服了 LinuxThreads的缺点，同时也符合 POSIX 的需求。与 LinuxThreads 相比，它在性能和稳定性方面都提供了重大的改进，一般都被包含在glibc 2.3.x里。以下是NPTL相对于LinuxThread实现的优点：
1. NPTL没有使用管理线程。管理线程的一些需求，例如向作为进程一部分的所有线程发送终止信号，是并不需要的；因为内核本身就可以实现这些功能。内核还会处理每个
线程堆栈所使用的内存的回收工作。它甚至还通过在清除父线程之前进行等待，从而实现对所有线程结束的管理，这样可以避免僵尸进程的问题。
2. 由于 NPTL 没有使用管理线程，因此其线程模型在 NUMA 和 SMP 系统上具有更好的可伸缩性和同步机制。
3. 使用 NPTL 线程库与新内核实现，就可以避免使用信号来对线程进行同步了。为了这个目的，NPTL 引入了一种名为 futex 的新机制。futex 在共享内存区域上进行工作，因此可以在进程之间进行共享，这样就可以提供进程间 POSIX 同步机制。我们也可以在进程之间共享一个 futex。这种行为使得进程间同步成为可能。实际上，NPTL 包含了一个 PTHREAD_PROCESS_SHARED 宏，使得开发人员可以让用户级进程在不同进程的线程之间共享互斥锁。
4. 由于 NPTL 是 POSIX 兼容的，因此它对信号的处理是按照每进程的原则进行的；getpid() 会为所有的线程返回相同的进程 ID。例如，如果发送了 SIGSTOP 信号，那么整个进程都会停止；使用 LinuxThreads，只有接收到这个信号的线程才会停止。这样可以在基于 NPTL 的应用程序上更好地利用调试器，例如 GDB。
5. 由于在 NPTL 中所有线程都具有一个父进程，因此对父进程汇报的资源使用情况（例如 CPU 和内存百分比）都是对整个进程进行统计的，而不是对一个线程进行统计的。
6. NPTL 线程库所引入的一个实现特性是对 ABI（应用程序二进制接口）的支持。
    
综观这些优点，其实大多都是依赖于新的内核对thread support。下面就这些特性，对kernel thread support方面一一阐述：

##Thread Group##

2.6.x kernel 里在进程管理上，使用了3个组的概念，它们分别是：组，线程组和会话组。普通的组，和传统的UNIX进程组概念一样：Modern Unix operating systems introduce the notion of process groups to represent a "job" abstraction。线程组则是新引入，支持POSIX标准的概念。会话组则和登陆终端有关。这些组关系紧密，并且会频繁的被kernel遍历查询，因此kernel引入一个hash表来连接这些组成员，以便快速的找到某个进程所属组的其他进程。一下是4个hash表的类型描述，不出意料，除了以上提到的3个组类型外，单个进程的pid也被加入到一个hash中。

**Table 3-5. The four hash tables and corresponding fields in the process descriptor**

| Hash table type | Field name | Description |
| --------------- | ------------| ------------- |
| PIDTYPE_PID | pid | PID of the process |
| PIDTYPE_TGID | tgid | PID of thread group leader process |
| PIDTYPE_PGID | pgrp | PID of the group leader process |
| PIDTYPE_SID | session | PID of the session leader process |

为支持该hash的使用，每个进程上下文结构task_struct中就加入了pid结构数组 struct pid pids[4]，大小为4，正好是四种类型。pid结构如下：

**Table 3-6. The fields of the pid data structures **
| Type | Name | Description |
| ------| ---- | -----------|
| int | nr | The PID number |
| struct hlist_node | pid_chain | The links to the next and previous elements in the hash chain list |
| struct list_head | pid_list | The head of the per-PID list |

- nr：进程id，但在TGID类型下，为thread group id
- pid_chain：hash冲突项列表，就是同一hash值下不同元素链表
- pid_list：连接同组内的进程

图1比较形象的描述了该hash。
由此，kernel其实已经有thread的概念了。在POSIX标准中，同线程组中的所有线程必须对有效信号进行处理，kernel就可以通过tgid（thread group id）可以找到所有的相关线程，在SIGKILL/SIGSTOP之类的信号发送到每个线程，而无需应用层去一一处理。关于信号会在下面详细介绍。


kernel要遍历某个线程所在线程组中的其他线程，调用next_thread：
#define pid_task(elem, type) \
list_entry(elem, struct task_struct, pids[type].pid_list)

task_t fastcall *next_thread(const task_t *p)
{
return pid_task(p->pids[PIDTYPE_TGID].pid_list.next, PIDTYPE_TGID);
}

通过pid获得task_struct结构：find_task_by_pid()
#define find_task_by_pid(nr) find_task_by_pid_type(PIDTYPE_PID, nr)

task_t *find_task_by_pid_type(int type, int nr)
{
struct pid *pid;

pid = find_pid(type, nr); /* hash 操作*/
if (!pid)
return NULL;

return pid_task(&pid->pid_list, type);
}

3）Futex

Futex在共享内存区域上进行工作，因此可以在进程之间进行共享，这样就可以提供进程间 POSIX 同步机制。Futex我还没仔细看过，貌似是对某个内存地址的临界访问。它提供wait/wake方法，通过syscall来调用。线程库可以利用futex实现semaphore和read/write lock。


4）Shared signal

有了thread group的概念，POSIX的标准就可以得到很好的支持。首先来看一下kill系统调用：
sys_kill(pid, sig)

pid > 0 向线程组id为pid的线程组发送sig消息。
pid = 0 向当前gid为pid的进程组中的所有线程组发送sig消息
pid = -1 发送sig给所有进程，swapper＆init除外
pid < -1 向当前gid为－pid的进程组的所有线程组发送sig消息
sys_kill 调用kill_something_info:
static int kill_something_info(int sig, struct siginfo *info, int pid)
{
if (!pid) {
return kill_pg_info(sig, info, process_group(current));
} else if (pid == -1) {
int retval = 0, count = 0;
struct task_struct * p;

read_lock(&tasklist_lock);
for_each_process(p) {
if (p->pid > 1 && p->tgid != current->tgid) {
int err = group_send_sig_info(sig, info, p);
++count;
if (err != -EPERM)
retval = err;
}
}
read_unlock(&tasklist_lock);
return count ? retval : -ESRCH;
} else if (pid < 0) {
return kill_pg_info(sig, info, -pid);
} else {
return kill_proc_info(sig, info, pid);
}
}
kill_pg_info()发送消息给组内的所有线程组。 kill_proc_info则发送给pid的线程组。
kill_pg_info()调用：
int __kill_pg_info(int sig, struct siginfo *info, pid_t pgrp)
{
struct task_struct *p = NULL;
int retval, success;

if (pgrp <= 0)
return -EINVAL;

success = 0;
retval = -ESRCH;
do_each_task_pid(pgrp, PIDTYPE_PGID, p) { /* 遍历hash中进程组中的所有进程 */
int err = group_send_sig_info(sig, info, p); /* 向进程的线程组发送消息 */
success |= !err;
retval = err;
} while_each_task_pid(pgrp, PIDTYPE_PGID, p);
return success ? 0 : retval;
}
kill_proc_info() 最终也调用group_send_sig_info去发送消息。至于详细的消息发送，可以研究研究group_send_sig_info。


5）/proc 实例

以往的kernel在/proc/下都会显示所有的进程id目录，现在只显示thread group id目录。当用户access /proc/目录时，一下函数被调用：
int proc_pid_readdir(struct file * filp, void * dirent, filldir_t filldir)
{
unsigned int tgid_array[PROC_MAXPIDS];
char buf[PROC_NUMBUF];
unsigned int nr = filp->f_pos – FIRST_PROCESS_ENTRY;
unsigned int nr_tgids, i;
int next_tgid;
….
for (;;) {
nr_tgids = get_tgid_list(nr, next_tgid, tgid_array); /* tgid_array 返回所有的tgid */
if (!nr_tgids) {
/* no more entries ! */
break;
}
next_tgid = 0;
… /* 创建tgid对应的inode节点… */
}
out:
return 0;
}

get_tgid_list函数获得所有的tgid，原型如下：
static int get_tgid_list(int index, unsigned long version, unsigned int *tgids)
{
struct task_struct *p;
int nr_tgids = 0;

index–;
read_lock(&tasklist_lock);
p = NULL;
…
if (p)
index = 0;
else
p = next_task(&init_task);

for ( ; p != &init_task; p = next_task(p)) {
int tgid = p->pid;
if (!pid_alive(p))
continue;
if (–index >= 0)
continue;
tgids[nr_tgids] = tgid;
nr_tgids++;
if (nr_tgids >= PROC_MAXPIDS)
break;
}
read_unlock(&tasklist_lock);
return nr_tgids;
}

最终调用next_task从init_task开始遍历获得所有tgid。因此在使用ps时，不加额外的参数，你可能看不到所有的线程，看到的只是线程组leader。那如何查看线程组中的所有线程？查看： /proc/TGID/task/
这时会调用get_tid_list为所有的线程创建inode：
static int get_tid_list(int index, unsigned int *tids, struct inode *dir)
{
struct task_struct *leader_task = proc_task(dir);
struct task_struct *task = leader_task;
int nr_tids = 0;

index -= 2;
read_lock(&tasklist_lock);
/*
* The starting point task (leader_task) might be an already
* unlinked task, which cannot be used to access the task-list
* via next_thread().
*/
if (pid_alive(task)) do {
int tid = task->pid;

if (–index >= 0)
continue;
if (tids != NULL)
tids[nr_tids] = tid;
nr_tids++;
if (nr_tids >= PROC_MAXPIDS)
break;
} while ((task = next_thread(task)) != leader_task);
read_unlock(&tasklist_lock);
return nr_tids;
}
最终调用next_thread遍历。别奇怪为什么linux老说线程是轻量级的进程，而ps看不到，线程是进程，不过是在kernel里被组织了起来。

    linux对线程的支持很妙，线程的创建和调度非常的高效。有了对kernel中支持线程的知识，相信平时在写pthread应用时会游刃有余。
# copy_process流程
1.检查clone_flags所传递的标志的一致性

	a.CLONE_NEWNS和CLONE_FS同时设置
	b.CLONE_THREAD设置为1，但CLONE_SIGHAND未设置
	c.CLONE_SIGHAND设置为1，但CLONE_VM未设置

2.调用security_task_create()以及稍后调用的security_task_alloc()执行所有附加的安全检查。

3.调用dup_task_struct()为子进程获取进程描述符，该函数执行如下操作：
	
	a.如果需要，则在当前进程中调用__unlazy_fpu()，把FPU,MMX和SSE/SSE2寄存器的内容保存到父进程的thread_info结构中。稍后，dup_task_struct()将把这些值复制到子进程的thread_info中
	b.调用alloc_task_struct宏，为新进程获取进程描述符，并将描述符地址保存在tsk局部变量中
	c.调用alloc_thread_info宏以获取一块空闲内存区，用来存放新进程的thread_info结构和内核栈，并将这块内存区字段的地址存在局部变量ti中。这块内存区字段大小一般是两页，即8KB
	d.将current进程描述符的内容复制到tsk所指向的task_struct结构中，然后把tsk->thread_info置为ti
	e.将current进程的thread_info描述符的内容复制到ti所指向的内容中，然后把ti->task置为tsk
	f.把新进程描述符的使用计数器(tsk->usage)置为2，用来表示进程描述符正在被使用而且其相应的进程处于活动状态（进程状态不是EXIT_ZOMBIE,也不是EXIT_DEAD)
	g.返回新进程的进程描述符指针(tsk)

4.检查存放在current->signal->rlim[RLIMIT_NPROC].rlim_cur变量中的值是否小于或等于用户所拥有的进程数。如果是，则返回错误码，除非进程没有root权限。该函数从user_struct中获取用户所拥有的进程数：current->user->processes。

5.递增user_struct结构的使用计数器(tsk->user->__count字段）和用户所拥有的进程的计数器(tsk->user->processes)。

6.检查系统中的进程数量（存放在nr_threads变量中）是否超过max_threads变量的值。这个变量的缺省值（默认值）取决于系统物理内存容量的大小。总的原则是，所有thread_info描述符和内核栈所占用的空间不能超过物理内存大小的1/8。不过，系统管理员可以通过写/proc/sys/kernel/threads-max文件来改变这个值。

7.设置与进程状态相关的几个字段：

	a.把大内核锁计数器tsk->lock_depth初始化为-1
	b.把tsk->did_exec字段初始化为0：它记录了进程发出的execve()系统调用的次数
	c.更新从父进程复制到tsk->flags字段中的一些标志：首先清楚PF_SUPERPRIV标志，该标志表示进程是否使用了某种超级用户权限。然后设置PF_FORKNOEXEC标志，它表示子进程还没有发出execve()系统调用

8.把新进程的PID存入tsk->pid字段

9.如果clone_flags参数中的CLONE_PARENT_SETTID标志被设置，就把子进程的PID复制到参数parent_tidptr指向的用户态变量中。

10.初始化子进程描述符中的list_head数据结构和自旋锁（这是存储在task_struct中的结构体，而不是外部结构体，所以不需要使用指针），并为与挂起信号、定时器及时间统计表相关的几个字段赋初值。

11.调用copy_semundo(),copy_files(),copy_fs(),copy_sighand(),copy_signal(),copy_mm()和copy_namespace()来创建新的数据结构，并把父进程current相应数据结构的值复制到新数据结构中，除非clone_flags参数指出它们有不同的值。

12.调用copy_thread()，用发出clone()系统调用时CPU寄存器的值（这些值已经被保存在父进程的内核栈中）来初始化子进程的内核栈。不过，copy_thread()把eax寄存器对应字段的值（这是fork()和clone()系统调用在子进程中的返回值)字段强行置为0。子进程描述符的thread.esp字段初始化为子进程内核栈的基地址，汇编函数(ret_from_fork())的地址存放在thread.eip字段中。如果父进程使用I/O权限位图，则子进程获取该位图的一个拷贝。最后，如果CLONE_SETTLS标志被设置，则子进程获取由clone()系统调用的参数tls指向的用户态数据结构所表示的TLS段

13.如果clone_flags参数的值被置为CLONE_CHILD_SETTID或CLONE_CHILD_CLEARTID，就把child_tidptr参数的值分别复制到tsk->set_child_tid中。这些标志说明：必须改变子进程用户态地址空间的child_tidptr所指向的变量的值，不过实际的写操作要稍后再执行。

14.清除子进程thread_info结构的TIF_SYSCALL_TRACE标志，以使ret_from_fork()函数不会把系统调用结束的消息通知给调试进程。（因为对子进程的跟踪是由tsk->ptrace中的PTRACE_SYSCALL标志来控制的，所以子进程的系统调用跟踪不会被禁用。

15.用clone_flags参数低位的信号数字编码初始化tsk->exit_signal字段，如果CLONE_THREAD标志被置位，就把tsk->exit_signal字段初始化为-1.只有当线程组的最后一个成员（通常是线程组的领头）“死亡”，才会产生一个信号，以通知线程组的领头线程的父进程

16.调用sched_fork()完成对新进程调度程序数据结构的初始化。该函数把新进程的状态设置为TASK_RUNNING,并把thread_info结构的preempt_count字段设置为1，从而禁止内核抢占。此外，歪了保证公平的进程调度，该函数再父子进程之间共享父进程的时间片。

17.把新进程的thread_info结构的cpu字段设置为由smp_processor_id()所返回的本地CPU号

18.初始化表示亲子关系的字段。尤其是，如果CLONE_PARENT或CLONE_THREAD被设置，就用current->real_parent的值初始化tsk->real_parent和tsk->parent，因此，子进程的父进程似乎是当前进程的父进程。否则，把tsk->real_parent和tsk->parent置为当前进程

19.如果不需要跟踪子进程（没有设置CLONE_PTRACE标志），就把tsk->ptrace字段设置为0.

20.执行SET_LINKS宏，把新进程描述符插入进程链表。

21.如果子进程必须被跟踪（tsk->ptrace字段的PT_PTRACED标志被设置），就把current->parent赋给tsk->parent，并将子进程插入调试程序的跟踪链表中。

22.调用attach_pid()把新进程描述符的PID插入pidhash[PIDTYPE_PID]散列表。

23.如果子进程是线程组的领头进程（CLONE_THREAD标志被清0）

	a.把tsk->tgid的初值置为tsk->pid
	b.把tsk->group_leader的初值置为tsk
	c.调用三次attach_pid()，把子进程PID分别插入PIDTYPE_TGID,PIDTYPE_PGID和PIDTYPE_SID类型的PID散列表

24.否则，如果子进程属于它的父进程的线程组（CLONE_THREAD标志被设置）

	a.把tsk->tgid的初值置为current->tgid
	b.把tsk->group_leader的初值置为current->group_leader
	c.调用attach_pid()，把子进程插入PIDTYPE_TGID类型的散列表中（插入current->group_leader进程的每个PID链表）

25.现在，新进程已经被加入进程集合：递增nr_threads变量的值

26.递增total_forks变量以记录被创建的进程的数量

27.终止并返回子进程描述符指针(tsk)
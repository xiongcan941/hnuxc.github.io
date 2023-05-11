---
layout:     post
title:      Linux kernel Notifier
subtitle:   Linux kernel Notifier
date:       2023-05-11
author:     xc
header-img: img/post-bg-ios9-web.jpg
catalog: true
tags:
    - Linux
    - kernel
    - Notifier
---

# Linux内核通知链(Notifier)原理及在我的项目中运用
## 为何我要学习Notifier
在合作的轻量级Linux完整性项目中，我有一个HOOK内核函数的内核模块 `hook.ko` 以及进行完整性度量 `integrity.ko` 的内核模块，每当  `hook.ko` 发现系统中发生可能造成系统完整性遭到破坏的行为，就需通知 `integrity.ko` 进行相应度量对象的度量；
 `hook.ko` 与 `integrity.ko` 之间如何进行通信，则需要Notifier的出场。

## Notifier原理

在Linux kernel中，各个内核子系统之间有很强的相互关系，它们之间若要相互通信，则需要一个通知机制，例如我上述的需求。Notifier机制只能用在内核子系统之间，内核与用户态之间不能使用。源码/kernel/notifier.c

Notifier实现（位于include/linux/notifier.h)：

```
这是linux kernel 5.10源码
struct notifier_block;

typedef	int (*notifier_fn_t)(struct notifier_block *nb,
			unsigned long action, void *data);//回调函数格式

struct notifier_block {
	notifier_fn_t notifier_call;//每个节点的回调函数
	struct notifier_block __rcu *next;//链表
	int priority;//优先级
};
```

阅读上述源码，可以明白`Notifier`的实现机制，把所有需要通信的模块连接到一个链表中，在链表的节点上注册自己的回调函数，每当链表之中有通知发出，每个节点接收到通知则会执行自己定义的回调函数，每个节点可通过设置priority来定义自己的优先级，在通知发生时，就可以进行按照优先级进行函数指针的回调操作。（这里不得不说linux kernel之中链表的作用真的很大，例如进程，内核模块等都通过链表连接，这里也不例外）

运用上述结构体，Linux为我们提供了四类notifier链

- **atomic_notifier**

```
这是linux kernel 5.10源码
struct atomic_notifier_head {
	spinlock_t lock;//自旋锁
	struct notifier_block __rcu *head;//上述链表结构体
};
``` 

可以看出原子通知链是对上述 `notifier_block` 的封装，在`notifier_block`基础上加上一个`自旋锁`，自旋锁是一种底层同步机制，只有两个值：“锁定”和“解锁”。在我看来，类似于互斥锁，它与互斥锁的区别在于，所有等待自旋锁的线程进入忙循环，而等待互斥锁的线程则进入休眠。正因为有这个特性，要求临界区代码必须短小以及持有自旋锁的时间尽可能短。

正因为封装了一个自旋锁，所以原子通知链每个节点的回调函数执行时需要申请自旋锁，执行完毕后释放自旋锁，所以回调函数需要运行在原子上下文，而且不能休眠。

__rcu：是地址空间的宏定义

```
/include/linux/compiler_types.h
attribute是用来修饰一个变量的，这个变量必须是no dereference
# define __kernel	__attribute__((address_space(0))) //指针地址必须在内核空间
# define __user		__attribute__((noderef, address_space(__user))) //指针地址必须在用户空间
# define __iomem	__attribute__((noderef, address_space(__iomem))) //指针地址必须在io存储空间
# define __percpu	__attribute__((noderef, address_space(__percpu))) //指针地址必须在cpu空间
# define __rcu		__attribute__((noderef, address_space(__rcu)))
RCU代表的是 “read, copy, update”。它是一种算法，允许多个读者访问数据，并且同时允许修改者，删除者能够进行操作。
使用__rcu 附上 RCU保护的数据结构，如果你没有使用rcu_dereference()类中某个函数，Sparse就会警告你这个操作。
```

- **blocking_notifier**

```
struct blocking_notifier_head {
	struct rw_semaphore rwsem;//读写信号量
	struct notifier_block __rcu *head;
};
```
读/写信号量适于在读多写少的情况下使用。如果一个任务需要读和写操作时，它将被看作写者，在不需要写操作的情况下可降级为读者。任意多个读者可同时拥有一个读/写信号量，对临界区代码进行操作。

在没有写者操作时，任何读者都可成功获得读/写信号量进行读操作。如果有写者在操作时，读者必须被挂起等待直到写者释放该信号量。在没有写者或读者操作时，写者必须等待前面的写者或读者释放该信号量后，才能访问临界区。写者独占临界区，排斥其他的写者和读者，而读者只排斥写者。

所以可阻塞通知链每个节点的回调函数在执行时申请读写信号量，若为写者，则阻塞其他读者与写者；若为读者，则阻塞写者，执行完毕后释放该信号量。rwsem可视作是rwlock的可睡眠版本。

- **raw_notifier**

```
struct raw_notifier_head {
	struct notifier_block __rcu *head;
};
```
这个与最开始的版本一致，所有保护机制由节点自己定义

- **srcu_notifier**

```
struct srcu_notifier_head {
	struct mutex mutex;
	struct srcu_struct srcu;
	struct notifier_block __rcu *head;
};采用SRCU(Sleepable Read-Copy Update)代替rw-semphore来保护chains
```

SRCU其实就是sleepable RCU的缩写，而我们常说的RCU实际上是classic RCU，也就是在reader critical section中不能睡眠的，其在临界区内的代码要求是spin lock一样的。也正因为如此，我们可以在进程调度的时候可以判断该CPU的QS已经通过。SRCU是一个RCU的变种，从名字上也可以看出来，其reader critical section中可以block。

## Notifier应用

### Notifier初始化

```
动态初始化
#define ATOMIC_INIT_NOTIFIER_HEAD(name) do {	\
		spin_lock_init(&(name)->lock);	\
		(name)->head = NULL;		\
	} while (0)
#define BLOCKING_INIT_NOTIFIER_HEAD(name) do {	\
		init_rwsem(&(name)->rwsem);	\
		(name)->head = NULL;		\
	} while (0)
#define RAW_INIT_NOTIFIER_HEAD(name) do {	\
		(name)->head = NULL;		\
	} while (0)
可以看到，我们的节点随着name的变化而变化
```

```
静态初始化
#define ATOMIC_NOTIFIER_INIT(name) {				\
		.lock = __SPIN_LOCK_UNLOCKED(name.lock),	\
		.head = NULL }
#define BLOCKING_NOTIFIER_INIT(name) {				\
		.rwsem = __RWSEM_INITIALIZER((name).rwsem),	\
		.head = NULL }
#define RAW_NOTIFIER_INIT(name)	{				\
		.head = NULL }

#define SRCU_NOTIFIER_INIT(name, pcpu)				\
	{							\
		.mutex = __MUTEX_INITIALIZER(name.mutex),	\
		.head = NULL,					\
		.srcu = __SRCU_STRUCT_INIT(name.srcu, pcpu),	\
	}

#define ATOMIC_NOTIFIER_HEAD(name)				\
	struct atomic_notifier_head name =			\
		ATOMIC_NOTIFIER_INIT(name)
#define BLOCKING_NOTIFIER_HEAD(name)				\
	struct blocking_notifier_head name =			\
		BLOCKING_NOTIFIER_INIT(name)
#define RAW_NOTIFIER_HEAD(name)					\
	struct raw_notifier_head name =				\
		RAW_NOTIFIER_INIT(name)
```
所以静态初始化方式为：

- AUOMIC_NOTIFIER_HEAD(headname)
- BLOCKING_NOTIFIER_HEAD(headname)
- RAW_NOTIFIER_HEAD(headname)

### Notifier节点注册/注销

```
static int notifier_chain_register(struct notifier_block **nl,
		struct notifier_block *n)
{
	while ((*nl) != NULL) {
		if (unlikely((*nl) == n)) {
			WARN(1, "double register detected");
			return 0;
		}
		if (n->priority > (*nl)->priority)
			break;
		nl = &((*nl)->next);
	}
	n->next = *nl;
	rcu_assign_pointer(*nl, n);
	return 0;
}

static int notifier_chain_unregister(struct notifier_block **nl,
		struct notifier_block *n)
{
	while ((*nl) != NULL) {//从头节点开始依次遍历
		if ((*nl) == n) {
			rcu_assign_pointer(*nl, n->next);
			return 0;
		}
		nl = &((*nl)->next);//从链表中找到该节点删除
	}
	return -ENOENT;
}
也就是从链表中加入或删除节点
```
其他几类的注册/注销函数随着封装的不同而改变
```
int atomic_notifier_chain_register(struct atomic_notifier_head *nh,struct notifier_block *n)
int atomic_notifier_chain_unregister(struct atomic_notifier_head *nh,struct notifier_block *n)

int blocking_notifier_chain_register(struct blocking_notifier_head *nh,struct notifier_block *n)
int blocking_notifier_chain_unregister(struct blocking_notifier_head *nh,struct notifier_block *n)

int raw_notifier_chain_register(struct raw_notifier_head *nh,struct notifier_block *n)
int raw_notifier_chain_unregister(struct raw_notifier_head *nh,struct notifier_block *n)

int srcu_notifier_chain_register(struct srcu_notifier_head *nh,struct notifier_block *n).
int srcu_notifier_chain_unregister(struct srcu_notifier_head *nh,struct notifier_block *n)
```

### Notifier的通知
```
static int notifier_call_chain(struct notifier_block **nl//链表头节点,
			       unsigned long val, void *v,
			       int nr_to_call, int *nr_calls)
{
	int ret = NOTIFY_DONE;
	struct notifier_block *nb, *next_nb;

	nb = rcu_dereference_raw(*nl);

	while (nb && nr_to_call) {//遍历链表，依次调用回调函数
		next_nb = rcu_dereference_raw(nb->next);

#ifdef CONFIG_DEBUG_NOTIFIERS
		if (unlikely(!func_ptr_is_kernel_text(nb->notifier_call))) {
			WARN(1, "Invalid notifier called!");
			nb = next_nb;
			continue;
		}
#endif
		ret = nb->notifier_call(nb, val, v); //调用注册的回调函数

		if (nr_calls)
			(*nr_calls)++;

		if (ret & NOTIFY_STOP_MASK)//有停止的mask就返回，否则继
			break;
		nb = next_nb;//走向下一个节点
		nr_to_call--;
	}
	return ret;
}
```
其他几类的通知函数随着封装的不同而改变
```
int atomic_notifier_call_chain(struct atomic_notifier_head *nh,unsigned long val, void *v)
int blocking_notifier_call_chain(struct blocking_notifier_head *nh,unsigned long val, void *v)
int raw_notifier_call_chain(struct raw_notifier_head *nh,unsigned long val, void *v)
int srcu_notifier_call_chain(struct srcu_notifier_head *nh,unsigned long val, void *v)
```

总的来说，通知的含义就是定义一个链表，将所有需要通知的模块加入到链表中，并且定义接收到通知的回调函数，发出通知就是遍历该链表，依次调用每个节点的回调函数，原理来说就是这么简单。注册/注销也就是从定义的通知链表之中加入或者删除节点

## 在我的项目中应用 

**hook.ko**
```
/*--------------------------notifier------------------------------------------*/
static RAW_NOTIFIER_HEAD(test_chain);

/*
 自定义的注册函数，将notifier_block节点加到刚刚定义的test_chain这个链表中来
*/
static int register_test_notifier(struct notifier_block *nb)
{
return raw_notifier_chain_register(&test_chain, nb);
}
EXPORT_SYMBOL(register_test_notifier);

static int unregister_test_notifier(struct notifier_block *nb)
{
return raw_notifier_chain_unregister(&test_chain, nb);
}
EXPORT_SYMBOL(unregister_test_notifier);

/*
* 自定义的通知链表的函数，即通知test_chain指向的链表中的所有节点执行相应的函数
*/
static int test_notifier_call_chain(unsigned long val, void *v)
{
return raw_notifier_call_chain(&test_chain, val, v);
}
/*--------------------------notifier------------------------------------------*/
```

**integrity.ko**
```
static int show(struct notifier_block *this, unsigned long event, void *ptr)
{
	tsk1 = kthread_run(show1(ptr), NULL, "mythread%d", 1);
	if (IS_ERR(tsk1)) {
		printk(KERN_INFO "create kthread failed!\n");
	}
	else {
		printk(KERN_INFO "create ktrhead ok!\n");
	}
	return 0;
}

extern int register_test_notifier(struct notifier_block*);
extern int unregister_test_notifier(struct notifier_block*);

static struct notifier_block test_notifier1 =
{
.notifier_call = show,
};
```
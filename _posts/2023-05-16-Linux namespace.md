---
layout:     post
title:      Linux namespace
subtitle:   Linux namespace
date:       2023-05-16
author:     xc
header-img: img/post-bg-ios9-web.jpg
catalog: true
tags:
    - Linux
    - kernel
---

# Linux namespace
## 计算机的虚拟化技术
操作系统通过`虚拟内存`技术，使得每个用户进程都认为自己拥有所有的物理内存，这是操作系统对内存的虚拟化。

操作系统通过`分时调度`系统，每个进程都能被[公平地]调度执行，即每个进程都能获取到CPU，使得每个进程都认为自己在进程活动期间拥有所有的CPU时间，这是操作系统对CPU的虚拟化。

从这两种**虚拟化方式**可推知，当使用某种虚拟化技术去管理进程时，进程会认为自己拥有某种物理资源的全部。

**然而**，虚拟内存和分时系统是对**物理资源**进行虚拟化，操作系统中还有许多**非物理资源**，比如用户权限系统资源，网络协议栈资源，文件系统挂载路径资源等。`Linux`通过`namespace`对这些**非物理资源**进行虚拟化。

Linux namespace是在当前运行的系统环境中(隔离)另一个进程的运行环境出来，并在此运行环境中将一些必要的系统全局资源进行**虚拟化**。进程运行在指定的namespace中，因此，namespace中的每个进程都认为自己拥有所有这些虚拟化的全局资源。

## namespace的形象化比喻
namespace，命名空间：打个比方，三年一班的小明和三年二班的小明，虽然他们名字相同，但是所在班级不同。那么，在全年级的排行榜上，即使出现两个名字一样的小明，也会通过不同班级区分。

对于学校来说，每个班级就相当于一个命名空间，这个空间的名称是班级号。

## namespace的用途与分类
从`docker`角度考虑如何实现一个资源隔离的容器：

 - 隔离文件系统：如利用chroot命令切换根目录的挂载点
 - 在分布式的环境下进行通信和定位：需拥有独立的IP、端口、路由等，这是对网络的隔离
 - 同时容器需要一个独立的主机名以便在网络中标识自己
 - 进程间的通信
 - 用户权限隔离
 - 运行在容器中的引用需要有自己的PID，需要与宿主机的PID隔离

所以由此Linux支持8种全局资源的虚拟化：
 
 - cgroup namespace:该namespace可单独管理自己的cgroup,即控制组，用于限制、控制、分离一个进程组的CPU,内存，网络，磁盘I/O等物理资源
 - ipc namespace：进程通信资源
 - network namespace：该namespace有自己的网络资源，包括网络协议栈、网络设备、路由表、防火墙、端口等
 - mount namespace：该namespace有自己的挂载信息，即拥有独立的目录层次
 - pid namespace：该namespace有自己的进程号，使得namespace中的进程PID单独编号，比如可以PID=1
 - time namespace：该namespace有自己的启动时间点信息和单调时间，比如可设置某个namespace的开机时间点为1年前启动，再比如不同的namespace创建后可能流逝的时间不一样
 - user namespace：该namespace有自己的用户权限管理机制(比如独立的UID/GID)，使得namespace更安全
 - uts namespace：该namepsace有自己的主机信息，包括主机名(hostname)、NIS domain name

用户可以同时创建具有多种资源类型的namespace，比如创建一个同时具有uts、pid和user的namespace。

## namespace介绍
命名空间建立系统的不同视图， 对于每一个命名空间，从用户看起来，应该像一台单独的Linux计算机一样，有自己的init进程(PID为0)，其他进程的PID依次递增，A和B空间都有PID为0的init进程，子容器的进程映射到父容器的进程上，父容器可以知道每一个子容器的运行状态，而子容器与子容器之间是隔离的。

Linux中有chroot的系统调用，该方法将进程限制到文件系统的某一部分，是一种简单的命名空间机制。

例如在task_struct结构体中，有struct nsproxy *nsproxy 这个成员变量

```
struct nsproxy {
	atomic_t count;
	struct uts_namespace *uts_ns;//Unix Timesharing System包含内存名称，版本，底层体系结构等信息
	struct ipc_namespace *ipc_ns;//保存所有与进程间通信（IPC）有关的信息
	struct mnt_namespace *mnt_ns;//当前挂载的文件系统
	struct pid_namespace *pid_ns_for_children;//有关进程id的信息
	struct net 	     *net_ns;//网络
	struct time_namespace *time_ns;
	struct time_namespace *time_ns_for_children;
	struct cgroup_namespace *cgroup_ns;
};
extern struct nsproxy init_nsproxy;
```

从以上可以看出，所谓的子空间，就是父进程通过fork得出子进程，子进程与父进程不共享某些资源，那么，这个子进程就处于自己的命名空间内。
要达到这种效果，就必须对fork的行为进行精准控制，内核提供如下参数来设置：

```
#define CLONE_NEWCGROUP		0x02000000	/* New cgroup namespace */
#define CLONE_NEWUTS		0x04000000	/* New utsname namespace */
#define CLONE_NEWIPC		0x08000000	/* New ipc namespace */
#define CLONE_NEWUSER		0x10000000	/* New user namespace */
#define CLONE_NEWPID		0x20000000	/* New pid namespace */
#define CLONE_NEWNET		0x40000000	/* New network namespace */
```

调用fork系统调用时，需检查上述标志位查询是否需新建命名空间

UTS命名空间

```
struct uts_namespace {
	struct kref kref;//引用计数器，用于跟踪内核中有多少地方使用uts_namespace的实例
	struct new_utsname name;//该namespace提供属性信息
	struct user_namespace *user_ns;//user_namespace包含用户id，组id，文件所有者id
	struct ucounts *ucounts;//
	struct ns_common ns;
} __randomize_layout;
extern struct uts_namespace init_uts_ns;
```

上述new_utsname

```
struct new_utsname {
	char sysname[__NEW_UTS_LEN + 1];//系统名称
	char nodename[__NEW_UTS_LEN + 1];
	char release[__NEW_UTS_LEN + 1];
	char version[__NEW_UTS_LEN + 1];
	char machine[__NEW_UTS_LEN + 1];
	char domainname[__NEW_UTS_LEN + 1];
};
```

使用uname -a可以查看这些信息
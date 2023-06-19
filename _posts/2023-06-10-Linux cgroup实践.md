---
layout:     post
title:      容器技术基石cgroup
subtitle:   cgroup
date:       2023-06-10
author:     xc
header-img: img/post-bg-ios9-web.jpg
catalog: true
tags:
    - linux
    - cgroup
    - namespace
---

# 容器技术基石
容器技术实现资源层面上的限制和隔离，依赖于Linux内核所提供的cgroup和namespace技术。

- cgroup的主要作用：管理资源的分配、限制；
- namespace的主要作用：封装抽象，限制，隔离，使命名空=间内的进程看起来拥有它们自己的全局资源；

# cgroup
cgroup是Linux内核的一个功能，用来限制、控制与分离一个进程组的资源（如CPU、内存、磁盘输入输出等）。cgroup 到目前为止，有两个大版本， cgroup v1 和 v2 。以下内容以 cgroup v2 版本为主，涉及两个版本差别的地方会在下文详细介绍。

cgroup 主要限制的资源是：

- CPU
- 内存
- 网络
- 磁盘 I/O

当我们将可用系统资源按特定百分比分配给 cgroup 时，剩余的资源可供系统上的其他 cgroup 或其他进程使用。

# cgroup文件结构

与cgroup v1类似，v2也是通过在cgroup相关目录下创建新的目录来创建cgroup控制对象的。

![](https://github.com/xiongcan941/xiongcan941.github.io/tree/master/img/1.JPG)

上图文件：

- cgroup.controllers:这个文件显示了当前cgroup可以限制的资源有哪些？这一个文件就可以控制当前cgroup支持哪些资源的限制。而不是像v1一样资源分别在不同的目录下创建相关cgroup。

```
xc@xc-virtual-machine:/sys/fs/cgroup$ cat cgroup.controllers
cpuset cpu io memory hugetlb pids rdma
```

表示根cgroup支持针对cpuset、cpu、io、memory、hugetlb、pids、rdma的限制。如果我们要创建可以支持更多资源限制能力的组，需要去上一级目录中的文件查看。

cgroup.subtree_control：这个文件内容应是cgroup.controllers的子集。其作用是限制在当前cgroup目录层级下创建的子目录中的cgroup.controllers内容。即子层级的cgroup资源限制范围被上一级的cgroup.subtree_control文件内容所限制。

```
xc@xc-virtual-machine:/sys/fs/cgroup$ cat cgroup.subtree_control
memory pids
```

可以看到，cgroup.subtree_control默认是对memory和pids进行限制。

若我们想创建一个可以支持cpu限制能力的cgroup组，应该做如下操作：

```
root@xc-virtual-machine:/sys/fs/cgroup# echo '+cpu' > cgroup.subtree_control
root@xc-virtual-machine:/sys/fs/cgroup# cat cgroup.subtree_control
cpu memory pids
root@xc-virtual-machine:/sys/fs/cgroup# mkdir my_cgroup
root@xc-virtual-machine:/sys/fs/cgroup# cd my_cgroup
root@xc-virtual-machine:/sys/fs/cgroup/my_cgroup# ls
cgroup.controllers  cgroup.max.descendants  cgroup.threads  cpu.stat        cpu.weight.nice  memory.events.local  memory.min        memory.stat          memory.swap.max
cgroup.events       cgroup.procs            cgroup.type     cpu.uclamp.max  io.pressure      memory.high          memory.numa_stat  memory.swap.current  pids.current
cgroup.freeze       cgroup.stat             cpu.max         cpu.uclamp.min  memory.current   memory.low           memory.oom.group  memory.swap.events   pids.events
cgroup.max.depth    cgroup.subtree_control  cpu.pressure    cpu.weight      memory.events    memory.max           memory.pressure   memory.swap.high     pids.max
root@xc-virtual-machine:/sys/fs/cgroup/my_cgroup# cat cgroup.controllers
cpu memory pids
```

此时我们创建的my_cgroup组就有cpu、memory、pids等常见的资源限制能力。另外要注意，被限制进程只能添加到叶子结点的组中，不能添加到中间结点的组中。

cgroup.events:包括两个只读的key-value。populated：1表示当前cgroup内有进程，0表示没有。frozen：1表示当前cgroup为frozen状态，0表示非此状态。

cgroup.type：表示当前cgroup的类型。"domain"：默认类型。"domain threaded":作为threaded类型cgroup的根结点。"domain invalid":无效状态cgroup。"threaded":threaded类型的cgroup。

这里所谓threaded模式其本质就是控制对象从进程为单位支持到了线程为单位。我们可以在一个由domain threaded类型的组中创建多个threaded类型的组，并把一个进程的多个线程放到不同的threaded类型组中进行资源限制。

```
domain:     进程子目录，控制进程资源的cgroup
threaded:   线程子目录，控制同一个进程下的线程资源
domain threaded:  线程子目录的根目录，threaded root
domain invalid:   线程子目录中处于无效状态的cgroup目录
```
cgroup.procs：这个文件显示的是当前这个cgroup中限制资源的进程ID。

cgroup.threads:当前这个cgroup中限制的线程ID。

cgroup.max.descendants：当前cgroup目录中可以允许的最大子cgroup个数。默认值为max。

cgroup.max.depth:当前cgroup目录中可以允许的最大cgroup层数。默认值为max。

cgroup.stat：包含两个只读的key-value。nr_descendants:当前cgroup下可见的子孙cgroup个数。nr_dying_desendants:这个cgroup下曾被创建但是已被删除的子孙cgroup个数。

cgroup.freeze：值为1可以将cgroup变为freeze状态，即被冻结。默认值为0。

## CPU资源隔离
与v1类似，针对cpu的限制仍然可以支持绑定核心、配额和权重。并且增加了压力通知机制，以便根据相关cpu压力作出相应反馈行为。例如当cpu压力达到一定程度就实现自动扩容。

### 绑定核心(cpuset)
使用cpuset资源隔离方式可以帮助我们把整个cgroup的进程都限定在某些cpu核心上运行，在numa架构下，还能帮我们绑定numa结点。

cpuset.cpus:用来制定当前cgroup绑定的cpu编号。如：

```
# cat cpuset.cpus
0-4,6,8-10
```

cpuset.cpus.effective:显示当前cgroup真实可用的cpu列表。

cpuset.mems:用来在numa架构上的服务器上绑定node结点。比如：

```
# cat cpuset.mems
0-1,3
```

cpuset.mems.effective:显示当前cgroup真实可用的mem node列表

### 配额

cpu.max：文件支持两个值，格式为： $MAX$PERIOD。

```
root@xc-virtual-machine:/sys/fs/cgroup/hellocgroup# cat cpu.max
max 100000
root@xc-virtual-machine:/sys/fs/cgroup/hellocgroup# echo 50000 1000000 > cpu.max
root@xc-virtual-machine:/sys/fs/cgroup/hellocgroup# cat cpu.max
50000 1000000
```
这里的含义是在CPU周期1秒内，有0.05秒是分给该cgroup的。也就是配置了该cgroup的cpu占用在单核上不超过5%。
```
xc@xc-virtual-machine:~/kernel_thread$ sudo insmod ttt.ko
root@xc-virtual-machine:/sys/fs/cgroup# cd my_cgroup
root@xc-virtual-machine:/sys/fs/cgroup/my_cgroup# echo 50000 1000000 > cpu.max
root@xc-virtual-machine:/sys/fs/cgroup/my_cgroup# echo threaded > cgroup.type
root@xc-virtual-machine:/sys/fs/cgroup/my_cgroup# echo 3611 > cgroup.threads 

 PID USER      PR  NI    VIRT    RES    SHR S  %CPU  %MEM     TIME+ COMMAND 
3611 root      20   0       0      0      0 D   4.0   0.0   0:05.60 mythrea+ 
  23 root      20   0       0      0      0 S   0.3   0.0   0:00.41 kcompac+ 
 2968 xc        20   0  579104  34872  18920 S   0.3   1.7   0:04.53 gnome-t+ 
 3980 xc        20   0   21876   3844   2988 R   0.3   0.2   0:00.04 top      
  1 root      20   0  166768   6424   3032 S   0.0   0.3   0:01.82 systemd  
```
可以看到无论如何我们的完整性度量线程mythread的CPU占用率都不会超过5%。

### 权重（cpuweight）
可以通过cpu.weight文件来设置本cgroup的权重值。默认为100。取值范围为[1， 10000]。

cpu.weight.nice：当前可以支持使用nice值的方式设置权重。取值范围和nice值范围一样[-20, 19]。

另外：

cpu.stat：是当前cgroup的cpu消耗统计。显示的内容包括：

usage_usec：占用cpu总时间。

user_usec：用户态占用时间。

system_usec：内核态占用时间。

nr_periods：周期计数。

nr_throttled：周期内的限制计数。

throttled_usec：限制执行的时间。

cpu.pressure：显示当前cgroup的cpu使用压力状态。详情参见：Documentation/accounting/psi.rst。psi是内核新加入的一种负载状态检测机制，可以目前可以针对cpu、memory、io的负载状态进行检测。通过设置，我们可以让psi在相关资源负载达到一定阈值的情况下给我们发送一个事件。用户态可以通过对文件事件的监控，实现针对相关负载作出相关相应行为的目的。psi的话题可以单独写一个文档，所以这里不细说了。

### 内存资源隔离
我们最常用也最好理解的就是对内存使用限制一个上限，应用程序使用不能超过此上限，超过就会退出，这就是硬限制。

memory.max：默认值为max，不限制。如果需要做限制，则写一个内存字节数上限到文件内就可以了。

memory.swap.max：使用swap的上限，默认为max。如果不想使用swap，设置此值为0。

memory.min：这是内存的硬保护机制。如果当前cgroup的内存使用量在min值以内，则任何情况下都不会对这部分内存进行回收。如果没有可用的不受保护的可回收内存，则将oom。这个值会受到上层cgroup的min限制影响，如果所有子一级的min限制总数大于上一级cgroup的min限制，当这些子一级cgroup都要使用申请内存的时候，其总量不能超过上一级cgroup的min。这种情况下，各个cgroup的受保护内存按照min值的比率分配。如果将min值设置的比你当前可用内存还大，可能将导致持续不断的oom。如果cgroup中没有进程，这个值将被忽略。

memory.current：显示当前cgroup内存使用总数。当然也包括其子孙cgroup。

memory.swap.current：显示当前cgroup的swap使用总数。

memory.high：内存使用的上限限制。与max不同，max会直接触发oom。而内存使用超出这个上限会让当前cgroup承受更多的内存回收压力。内核会尽量使用各种手段回收内存，保持内存使用减少到memory.high限制以下。

memory.low：cgroup内存使用如果低于这个值，则内存将尽量不被回收。这是一种是尽力而为的内存保护，这是“软保证”，如果cgroup及其所有子代均低于此阈值，除非无法从任何未受保护的cgroup回收内存，否则不会回收cgroup的内存。

memory.oom.group：默认值为0，值为1之后在内存超限发生oom的时候，会将整个cgroup内的所有进程都干掉，oom_score_adj设置为-1000的除外。

memory.stat：类似meminfo的更详细的内存使用信息统计。

memory.events：跟内存限制的相关事件触发次数统计，包括了所有子一级cgroup的相关统计。

memory.events.local：跟上一个一样，但是只统计自己的（不包含其他子一级cgroup）。

memory.swap.events：根swap限制相关的事件触发次数统计。

以上events文件在发生相关值变化的时候都会触发一个io事件，可以使用poll或select来接收并处理这些事件，已实现各种事件的上层相应机制。

memory.pressure：当前cgroup内存使用的psi接口文件。

### IO资源隔离
io资源隔离相比cgroup v1的改进亮点就是实现了buffer io的限制，让io限速使用在生产环境的条件真正成熟了。我们先来看一下效果：
```
[root@localhost zorro]# df
Filesystem                              1K-blocks     Used Available Use% Mounted on
devtmpfs                                   980892        0    980892   0% /dev
tmpfs                                     1000056        0   1000056   0% /dev/shm
tmpfs                                     1000056     1296    998760   1% /run
/dev/mapper/fedora_localhost--live-root  66715048 28671356  34611656  46% /
tmpfs                                     1000056        4   1000052   1% /tmp
/dev/mapper/fedora_localhost--live-home  32699156  2726884  28288204   9% /home
/dev/sda1                                  999320   260444    670064  28% /boot
tmpfs                                      200008        0    200008   0% /run/user/1000
[root@localhost zorro]# ls -l /dev/mapper/fedora_localhost--live-root
lrwxrwxrwx. 1 root root 7 Apr 14 13:53 /dev/mapper/fedora_localhost--live-root -> ../dm-0
[root@localhost zorro]# ls -l /dev/dm-0
brw-rw----. 1 root disk 253, 0 Apr 14 13:53 /dev/dm-0
[root@localhost zorro]# echo "253:0 wbps=2097152" > /sys/fs/cgroup/zorro/io.max
[root@localhost zorro]# cat !$
cat /sys/fs/cgroup/zorro/io.max
253:0 rbps=max wbps=2097152 riops=max wiops=max
```
按照上面的配置，我们就实现了 / 分区设置了一个2m/s的写入限速。
```
[root@localhost zorro]# cat dd.sh
#!/bin/bash

echo $$ > /sys/fs/cgroup/zorro/cgroup.procs
dd if=/dev/zero of=/bigfile bs=1M count=200
[root@localhost zorro]# ./dd.sh
200+0 records in
200+0 records out
209715200 bytes (210 MB, 200 MiB) copied, 0.208817 s, 1.0 GB/s
```
我们会发现，这时dd很快就把数据写到了缓存里。这里要看到限速效果，需要同时通过iostat监控针对块设备的写入：
```
avg-cpu:  %user   %nice %system %iowait  %steal   %idle
           0.00    0.00    0.25    2.24    0.00   97.51

Device             tps    kB_read/s    kB_wrtn/s    kB_dscd/s    kB_read    kB_wrtn    kB_dscd
dm-0             22.00         0.00      2172.00         0.00          0       2172          0
dm-1              0.00         0.00         0.00         0.00          0          0          0
dm-2              0.00         0.00         0.00         0.00          0          0          0
sda              15.00         0.00      2172.00         0.00          0       2172          0
sdb               0.00         0.00         0.00         0.00          0          0          0
scd0              0.00         0.00         0.00         0.00          0          0          0


avg-cpu:  %user   %nice %system %iowait  %steal   %idle
           0.00    0.00    0.00   23.44    0.00   76.56

Device             tps    kB_read/s    kB_wrtn/s    kB_dscd/s    kB_read    kB_wrtn    kB_dscd
dm-0             14.00         0.00      2080.00         0.00          0       2080          0
dm-1              0.00         0.00         0.00         0.00          0          0          0
dm-2              0.00         0.00         0.00         0.00          0          0          0
sda              14.00         0.00      2080.00         0.00          0       2080          0
sdb               0.00         0.00         0.00         0.00          0          0          0
scd0              0.00         0.00         0.00         0.00          0          0          0


avg-cpu:  %user   %nice %system %iowait  %steal   %idle
           0.00    0.00    0.25   21.70    0.00   78.05

Device             tps    kB_read/s    kB_wrtn/s    kB_dscd/s    kB_read    kB_wrtn    kB_dscd
dm-0             14.00         0.00      2052.00         0.00          0       2052          0
dm-1              0.00         0.00         0.00         0.00          0          0          0
dm-2              0.00         0.00         0.00         0.00          0          0          0
sda              14.00         0.00      2052.00         0.00          0       2052          0
sdb               0.00         0.00         0.00         0.00          0          0          0
scd0              0.00         0.00         0.00         0.00          0          0          0
```
命令执行期间，我们会发现iostat中，针对设备的write被限制在了2m/s。

除此之外，标记了direct的io事件限速效果根之前一样：
```
[root@localhost zorro]# cat dd.sh
#!/bin/bash

echo $$ > /sys/fs/cgroup/zorro/cgroup.procs
dd if=/dev/zero of=/bigfile bs=1M count=200 oflag=direct

[root@localhost zorro]# ./dd.sh
200+0 records in
200+0 records out
209715200 bytes (210 MB, 200 MiB) copied, 100.007 s, 2.1 MB/s
```
然后我们来看一下io的相关配置文件：
```
io.max：我们刚才已经使用了这个文件进行了写速率限制，wbps。除此以外，还支持rbps：读速率限制。riops：读iops限制。wiops：写iops限制。在一条命令中可以写多个限制，比如：

echo “8:16 rbps=2097152 wiops=120” > io.max
```
命令中的其他概念相信大家都明白了，不再多说了。

io.stat：查看本cgroup的io相关信息统计。包括：
```
          ======        =====================
          rbytes        Bytes read
          wbytes        Bytes written
          rios          Number of read IOs
          wios          Number of write IOs
          dbytes        Bytes discarded
          dios          Number of discard IOs
          ======        =====================
```
io.weight：权重方式分配io资源的接口。默认为：default 100。default可以替换成$MAJ:$MIN表示的设备编号，如：8:0

，表示针对那个设备的配置。后面的100表示权重，取值范围是：[1, 10000]。表示本cgroup中的进程使用某个设备的io权重是多少？如果有多个cgroup同时争抢一个设备的io使用的话，他们将按权重进行io资源分配。

io.bfq.weight：针对bfq的权重配置文件。

io.latency：这是cgroup v2实现的一种对io负载保护的机制。可以给一个磁盘设置一个预期延时目标，比如：
```
[root@localhost zorro]# echo "253:0 target=100" > /sys/fs/cgroup/zorro/io.latency
[root@localhost zorro]# cat !$
cat /sys/fs/cgroup/zorro/io.latency
253:0 target=100
```
target的单位是ms。如果cgroup检测到当前cgroup内的io响应延迟时间超过了这个target，那么cgroup可能会限制同一个父级cgroup下的其他同级别cgroup的io负载，以尽量让当前cgroup的target达到预期。更详细文档可以查看：Documentation/admin-guide/cgroup-v2.rst

io.pressure：当前cgroup的io资源的psi接口文件。

PIDS隔离
pids.max：限制当前cgroup内的进程个数。

pids.current：显示当前cgroup中的进程个数。包括其子孙cgroup。

最后
以上是cgroup v2的配置说明。我们会发现，跟v1相比，新版cgroup配置上的复杂度要小很多。并且加入了包括buffer io限制和psi等新的功能。新版cgroup也放弃了配置网络资源隔离的接口，当然需要的话，网络资源隔离部分还是可以直接使用tc进行配置。




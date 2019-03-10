---
layout:     post
title:      Linux性能优化(一)
subtitle:   "如何理解平均负载"
date:       2019-03-10
author:     Flint
header-img: img/average-load.jpg
catalog: true
tags:
    - Linux

---

# 如何查看平均负载

## uptime

我们可以使用下面的命令来查看系统的平均负载
```shell
# uptime
 12:43:01 up 30 days, 15:30,  2 users,  load average: 0.12, 0.04, 0.05
```

uptime命令输出结果的含义为

```shell
12:43:01						// 表示当前系统时间
up 30 days, 15:30				// 表示系统已经运行时间
2 users							// 表示当前登录用户数为2
load average: 0.12, 0.04, 0.05 	// 后面三个数字分别表示过去1分钟、5分钟、15分钟的平均负载
```

## 查询/proc/loadavg

```shell
# cat /proc/loadavg
0.00 0.06 0.06 3/167 10336
```

上述结果含义为

```shell
0.00 0.06 0.06					// 前面三个数字与uptime命令得到的load average是一致的
3/167							// 分母表示系统进程总数，分子表示正在运行的进程数
10336							// 表示最近运行的进程Id
```

# 平均负载到底是什么？

> 简单来说， 平均负载时指单位时间内，系统处于**可运行状态**和**不可中断状态**的平均进程数，也就是**平均活跃进程数**，它和CPU使用率并没有直接关系。

## Linux系统状态

Linux系统有五种状态：

- 可运行状态：是指正在CPU或者正在等待CPU的进程。也就是使用**ps -aux**命令下看到的处于**R**状态(Running或者Runnable)的进程。
- 睡眠状态：休眠中，进程在等待事件的完成。也就是**S**状态的进程。
- 不可中断状态：是指处于内核态关键流程中的进程，并且这些流程是不可中断的，比如最常见的就是等待硬件设备的I/O响应。也就是**D**状态(Disk Sleep)的进程。
- 僵尸状态：进程已经终止，但进程的描述符存在，直到父进程调用wait或waitpid系统调用后释放 。也就是**Z**状态。
- 停止状态：进程收到SIGSTOP，SIGSTP，SIGTIN，SIGTOU信号停止运行。也就是**T**状态。

> 1. 不可中断状态实际上是系统对进程和硬件设备的一种保护机制。比如一个进程向磁盘读写数据时候，为了保证数据的一致性，在得到磁盘的回复之前，它是不能被其他进程或者中断打断的。
> 2. Linux系统其他状态还包括**W**（无驻留页），**<**（高优先级进程），**N**（低优先级进程），**L**（内存锁页） 

## 平均负载意味着什么？

既然平均负载时平均活跃进程数，那么最理想的状态当然是每个CPU上刚好运行一个进程，这样每个CPU都会充分利用。比如当平均负载为2时候，意味着什么呢？

- 在只有2个CPU的系统上，意味着所有的CPU都刚好被完全占用。
- 在4个CPU的系统上，意味着CPU有50%的空闲。
- 在只有1个CPU的系统上，意味着有一个的进程竞争不到CPU。

## 平均负载设置多少合理？

我们知道相同的平均负载在CPU核数(这里指的是逻辑核数)不同的机器上的意义是完全不同的。所以，我们首先要知道CPU核数多少。

### top命令获取

1. 首先执行**top**命令
```shell
top - 13:25:00 up 30 days, 16:12,  2 users,  load average: 0.06, 0.15, 0.11
Tasks:  86 total,   2 running,  84 sleeping,   0 stopped,   0 zombie
%Cpu(s):  2.3 us,  3.0 sy,  0.0 ni, 94.6 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
```
2. 在**top**命令的显示界面，按数字**1**，即可查看当前系统中的总CPU核数。如下图就是为1核的CPU。
```shell
top - 13:27:25 up 30 days, 16:14,  2 users,  load average: 0.07, 0.16, 0.13
Tasks:  86 total,   2 running,  84 sleeping,   0 stopped,   0 zombie
%Cpu0  :  1.7 us,  1.3 sy,  0.0 ni, 97.0 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
```

### 查询/proc/cpuinfo

1. grep命令
```shell
# grep 'model name' /proc/cpuinfo
model name      : Intel(R) Xeon(R) Platinum 8163 CPU @ 2.50GHz
```

2. wc用法
Linux系统的wc(Word Count)命令的功能就是用于统计文件中的字节数、字数、行数，并将统计结果输出。
> - -c	         统计字节数
> - -l                 统计行数
> - -m              统计字符数，不能与-c一起使用
> - -w               统计字数，一个字被定义为由空白、跳格或者换行字符分隔的字符串
> - -L                打印最长行的长度
> - -help          显示帮助信息
> - —version  显示版本信息

3. 统计cpu核数
```shell
# grep 'model name' /proc/cpuinfo | wc -l
1
```

## 平均负载和CPU使用率的区别

- CPU密集型，使用大量的CPU会导致平均负载升高，这时候两者是一致的。
- IO密集型，等待I/O也会导致平均负载升高，但是CPU使用率不一定很高。
- 大量等待的进程调度也会导致平均负载升高，此时的CPU使用率也会比较高。

# 平均负载案例分析

## stress工具

stress工具是Linux系统下面压力测试工具，这里我们用作异常进程模拟平均负载升高的场景。

**Centos下安装**

```shell
# yum install stress
```

**stress参数说明**

```shell
 -?, --help         show this help statement 		显示帮助信息
     --version      show version statement	 		显示版本信息
 -v, --verbose      be verbose				 		显示运行信息
 -q, --quiet        be quiet				 		不显示运行信息
 -n, --dry-run      show what would have been done	显示已经完成的指令情况
 -t, --timeout N    timeout after N seconds			指定运行N秒后停止
     --backoff N    wait factor of N microseconds before work starts	等待N微秒后进程运行
 -c, --cpu N        spawn N workers spinning on sqrt()		产生n个进程 每个进程反复调用sqrt()
 -i, --io N         spawn N workers spinning on sync()		产生n个进程 每个进程反复调用sync()
 -m, --vm N         spawn N workers spinning on malloc()/free()	产生n个进程 malloc/free反复调用
     --vm-bytes B   malloc B bytes per vm worker (default is 256MB)	
     --vm-stride B  touch a byte every B bytes (default is 4096)
     --vm-hang N    sleep N secs before free (default none, 0 is inf)
     --vm-keep      redirty memory instead of freeing and reallocating
 -d, --hdd N        spawn N workers spinning on write()/unlink() 产生N个执行write和unlink函数的进程
     --hdd-bytes B  write B bytes per hdd worker (default is 1GB)
```

## sysstat工具

sysstat包含了常用的Linux性能工具，用于监控与分析系统的性能。

- mpstat是一个常用的多核CPU性能分析工具，用来实时查看每个CPU的性能指标，以及所有CPU的平均指标。
- pidstat是一个常用的进程性能分析工具，用来实时查看进程的CPU、内存、I/O以及上下文切换等性能指标

**安装**

1. 方式1：不推荐，因为不能安装最新的版本

```shell
# yum install systat
```

2. 方式2：使用rpm

```shell
# wget -c http://pagesperso-orange.fr/sebastien.godard/sysstat-12.1.3-1.x86_64.rpm
# rpm -Uvh sysstat-12.1.3-1.x86_64.rpm
```

**常用参数**

```shell
-u	# 默认的参数，显示各个进程的CPU使用统计
-r  # 显示各个进程内的内存使用统计
-d  # 显示各个进程内的IO使用情况
-p  # 指定进程号
-w  # 显示每个进程的上下文切换情况
-t  # 显示选择任务的统计信息外的额外信息
```

## 场景一：CPU密集进程

1. term1模拟一个CPU使用率100%的场景

```shell
# stress -c 1 -t 600
```

2. term2运行uptime查看平均负载情况

```shell
# watch -d uptime
 15:25:34 up 30 days, 18:12,  8 users,  load average: 3.09, 1.75, 0.74
```

> watch可以帮助检测一个命令的运行结果。
>
> -n, —interval  		  # 用来执行命令运行间隔，默认两秒
>
> -d, —differences	     # 会高亮显示变化的区域
>
> -t, -no-title			# 不显示watch在顶部的时间间隔，命令以及当前时间

3. term3运行mpstat查看CPU使用率的变化情况

```shell
# mpstat -P ALL 5 1
Linux 3.10.0-957.1.3.el7.x86_64 (izuf6dgf358qnu4kegmelzz)       03/10/2019      _x86_64_        (1 CPU)

03:53:57 PM  CPU    %usr   %nice    %sys %iowait    %irq   %soft  %steal  %guest  %gnice   %idle
03:54:02 PM  all   89.32    0.00   10.46    0.00    0.00    0.22    0.00    0.00    0.00    0.00
03:54:02 PM    0   89.32    0.00   10.46    0.00    0.00    0.22    0.00    0.00    0.00    0.00
```

> -P ALL表示监控所有的CPU，5表示间隔5秒后输出一组数据

**从term2可以看出1分钟内的平均负载增加了，从term3可以看出CPU的使用率89%，但它的iowait只有0，这说明平均负载的升高正是由于CPU使用率接近100%。**

4. term4运行pidstat

```shell
# pidstat -u 5 1
Linux 3.10.0-957.1.3.el7.x86_64 (izuf6dgf358qnu4kegmelzz)       03/10/2019      _x86_64_        (1 CPU)

03:53:30 PM   UID       PID    %usr %system  %guest   %wait    %CPU   CPU  Command
03:53:35 PM     0      3028    0.20    0.20    0.00    0.00    0.39     0  containerd
03:53:35 PM     0     14185    0.20    0.20    0.00    1.18    0.39     0  sshd
03:53:35 PM     0     22621    0.00    0.20    0.00    0.98    0.20     0  sshd
03:53:35 PM     0     22738    0.20    0.20    0.00    0.20    0.39     0  top
03:53:35 PM     0     24349    0.20    0.20    0.00    0.39    0.39     0  watch
03:53:35 PM     0     27738    0.20    0.20    0.00    0.20    0.39     0  top
03:53:35 PM     0     29639   89.17    0.20    0.00   10.43   89.37     0  stress
03:53:35 PM     0     30560    0.00    0.20    0.00    0.20    0.20     0  pidstat
```

**这里很明显可以看出stress进程导致的CPU使用率过高。**

## 场景二： I/O密集型

1. term1

```shell
# stress -i 1 -t 600
```

2. term2

```shell
# watch -d uptime
 15:41:12 up 30 days, 18:28,  8 users,  load average: 2.33, 1.75, 1.40
```

3. term3

```shell
# mpstat -P ALL 5 1
Linux 3.10.0-957.1.3.el7.x86_64 (izuf6dgf358qnu4kegmelzz)       03/10/2019      _x86_64_        (1 CPU)

03:41:29 PM  CPU    %usr   %nice    %sys %iowait    %irq   %soft  %steal  %guest  %gnice   %idle
03:41:34 PM  all    5.56    0.00   65.84   28.40    0.00    0.21    0.00    0.00    0.00    0.00
03:41:34 PM    0    5.56    0.00   65.84   28.40    0.00    0.21    0.00    0.00    0.00    0.00
```

4. term4

```shell
# pidstat -u 5 1
Linux 3.10.0-957.1.3.el7.x86_64 (izuf6dgf358qnu4kegmelzz)       03/10/2019      _x86_64_        (1 CPU)

03:41:41 PM   UID       PID    %usr %system  %guest   %wait    %CPU   CPU  Command
03:41:46 PM     0        42    0.00    1.00    0.00    7.39    1.00     0  kworker/u2:1
03:41:46 PM     0      1172    0.00    3.59    0.00    0.00    3.59     0  kworker/0:1H
03:41:46 PM     0      2290    0.20    0.00    0.00    0.40    0.20     0  auditd
03:41:46 PM     0      2786    0.20    0.00    0.00    0.00    0.20     0  dockerd
03:41:46 PM     0      3028    0.00    0.20    0.00    0.00    0.20     0  containerd
03:41:46 PM   996      4195    0.20    0.00    0.00    0.00    0.20     0  node
03:41:46 PM     0     14185    0.20    0.00    0.00    0.40    0.20     0  sshd
03:41:46 PM     0     15103    0.40   47.31    0.00    9.18   47.70     0  stress
03:41:46 PM     0     22621    0.00    0.20    0.00    1.00    0.20     0  sshd
03:41:46 PM     0     22815    0.20    0.20    0.00    0.60    0.40     0  sshd
03:41:46 PM     0     24349    0.20    0.40    0.00    0.40    0.60     0  watch
03:41:46 PM     0     27577    0.40    0.00    0.00    0.60    0.40     0  sshd
03:41:46 PM     0     27738    0.20    0.20    0.00    0.20    0.40     0  top
03:41:46 PM     0     28985    0.00    0.20    0.00    0.00    0.20     0  pidstat
```

**可以看到平均负载升高了，但是CPU使用率不到6%，而iowait达到28.4%。这说明平均负载的升高是由于iowait升高导致的。**

## 场景三：大量进程

1. term1

```shell
# stress -c 8 -t 600
```

2. term2

```shell
# watch -d uptime
15:49:22 up 30 days, 18:36,  6 users,  load average: 10.11, 5.21, 2.85
```

3. term3

```shell
# pidstat -u 5 1
Linux 3.10.0-957.1.3.el7.x86_64 (izuf6dgf358qnu4kegmelzz)       03/10/2019      _x86_64_        (1 CPU)

03:49:08 PM   UID       PID    %usr %system  %guest   %wait    %CPU   CPU  Command
03:49:13 PM     0       103    0.00    0.20    0.00    3.35    0.20     0  kauditd
03:49:13 PM     0      1306    0.20    0.00    0.00    0.20    0.20     0  systemd-journal
03:49:13 PM     0      2835    0.20    0.00    0.00    0.00    0.20     0  containerd
03:49:13 PM     0      3028    0.20    0.20    0.00    0.00    0.39     0  containerd
03:49:13 PM   996      4195    0.20    0.00    0.00    0.20    0.20     0  node
03:49:13 PM     0     14185    0.20    0.20    0.00    0.99    0.39     0  sshd
03:49:13 PM     0     14255    0.20    0.20    0.00    1.18    0.39     0  top
03:49:13 PM     0     18469   10.65    0.00    0.00   88.56   10.65     0  stress
03:49:13 PM     0     18470   10.65    0.00    0.00   88.76   10.65     0  stress
03:49:13 PM     0     18471   10.65    0.00    0.00   87.97   10.65     0  stress
03:49:13 PM     0     18472   10.45    0.00    0.00   89.35   10.45     0  stress
03:49:13 PM     0     18473   10.65    0.00    0.00   89.35   10.65     0  stress
03:49:13 PM     0     18474   10.45    0.00    0.00   87.77   10.45     0  stress
03:49:13 PM     0     18475   10.65    0.20    0.00  121.89   10.85     0  stress
03:49:13 PM     0     18476   12.82    0.00    0.00   73.18   12.82     0  stress
03:49:13 PM     0     21798    0.00    0.20    0.00    1.97    0.20     0  pidstat
03:49:13 PM     0     22621    0.20    0.00    0.00    0.99    0.20     0  sshd
03:49:13 PM     0     24349    0.39    0.20    0.00    1.38    0.59     0  watch
03:49:13 PM     0     27577    0.20    0.00    0.00    0.79    0.20     0  sshd
03:49:13 PM     0     27738    0.00    0.20    0.00    0.59    0.20     0  top
```

**可以看出系统的CPU处于严重过载情况，8个进程抢夺1个CPU，每个进程等待CPU的时间（也就是%wait列）高达89%，这些超过CPU计算能力的进程，最终导致CPU过载。**
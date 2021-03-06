---
title: Java进程被杀死排查过程
date: 2020-11-03 23:46:11
tags: Linux
categories: 技术

---



# 相关知识

Linux  `oom-killer`	是一种自我保护机制，当系统分配不出内存时(触发条件)会触发这个机制，由操作系统在己有进程中挑选一个占用内存较多，回收内存收益最大的进程kill掉来释放内存。系统为每个进程做评估(/proc/<pid>/oom_score中数值最大的进程被kill掉。

当发生oom的时候，可以记录在`/var/log/messages`中，如下：

![oom-killer](https://newhcw.github.io/images/oomkiller.png)



# 排查思路

1. 推测原因：进程被kill可能的原因有哪些？
   - 被人为的 kill -9 pid了，可能性不大，暂不讨论。
   - 操作系统内存不足，触发了oom-killer机制，自动杀死了
2. 验证猜测：
   - 查看操作系统日志`/var/log/messages`错误日志



# 排查步骤

1. 查看系统是否宕机或者重启，寻找关键词`kmsg started`，在xshell输入

   ```shell
   # cat /var/log/messages
   ```

   执行结果如下：

   ![linuxrestart](https://newhcw.github.io/images/linuxrestart.jpeg)

   显示结果2020-11-13 10:16有输出`kmsg started`，即发生过系统重启，这是人为重启，还是操作系统重启呢，我还不能知道。

   2. less一下顺着日志查找异常信息

![oom-killer2](https://newhcw.github.io/images/oom-kiiler2.jpeg)

发现了开头提到的关键词`oom killer`,根据之前的知识，说明系统曾经内存不足，linux将一些进程杀死了，这就解释了为什么Java进程会被kill。

回到之前的问题，那么重启是什么导致的呢？

因为是开发机器，使用的人比较多，我猜测是系统内存爆满导致系统不可用，所以有人重启了系统。

还有一种可能，操作系统杀死占内存大的进程后，依然不够申请多余的内存，就会自动重启系统。


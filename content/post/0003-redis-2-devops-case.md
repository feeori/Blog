---
title: "Redis：两则线上运维经验"
date: 2021-03-20T20:22:49+08:00
categories: ["线上运维"]
tags: ["Redis"]
draft: false
---
## 前言
Redis作为缓存，广泛应用于各种互联网业务。Redis本身设计优美，实现简单，绝大多数情况下，稳定运行，很少碰到故障。但线上场景多种多样，还是会碰到一些棘手的需求或问题，在此，记录一下曾遇到过的两个线上问题。

## Redis在线迁移
### 迁移工具redis-migrate-tool
Redis迁移有不少成功案例，一般使用唯品会开源的[redis-migrate-tool](https://github.com/vipshop/redis-migrate-tool)，该工具通过如下流程迁移数据：模拟Slave->得到源Master的数据并解析成Redis命令->写入到目标Redis。工具还提供校验功能，历经考验，稳定好用。

### 迁移需求
本次迁移是从单机版Redis向集群版Redis迁移，通过redis-migrate-tool工具即可实现。但业务的需求比较特殊，业务使用Redis用来存放`分布式锁`，而非常见的数据缓存。分布式锁对一致性的要求非常高，迁移过程中，如何能保证一致性不被破坏，又不影响正常的业务，让业务无感知，是本次迁移的难点。

### 迁移流程
最终迁移的思路为：
* 分阶段进行迁移。通过设置一个特殊的Redis键，来触发/控制阶段的切换；
* 设置一个特殊的`拦截阶段`，在该阶段内，所有`写`分布式锁的业务代码，sleep 200ms（sleep时间需实测。确保源Redis数据改变后，在sleep的时间内，改变的数据能够全部同步到目的Redis集群）

迁移前，业务需要修改代码，找到所有写分布式锁的操作，并加入sleep，一旦Redis中设置了触发键，`写操作`需要sleep 200ms。架构图与具体迁移阶段如下：
```
-------------  redis-migrate-tool   -------------
| 单点Redis | --------------------> | 集群Redis |
-------------                       -------------
           \                        /
            \                      /
             \                    /
              \                  /
               \                /
                ----------------
                | 业务java程序 |
                ----------------
```
* 准备阶段
    - 业务需要修改代码，在所有写操作前，加入sleep代码；
    - 迁移前，运行redis-migrate-tool，确保存量数据全部同步完成。保证进入初始阶段时，仅在同步增量数据；
* 初始阶段
    - 业务代码读写原始的单点Redis；
* 双读阶段
    - 业务代码先去目的Redis集群读，如果目的集群读不到，再去原单点Redis读，并统计目的集群读不到的比例，用于验证同步工具的时效性（即增量的数据是否及时同步到目的集群了）。
	- 注意，此时写操作，还是写入原Redis；
* `拦截阶段`
    - 该阶段所有`写锁的操作`，都会被sleep 200ms。保证同步工具把锁同步到目的集群；
	- sleep完成后，写操作写向目的Redis集群；
	- 读操作，优先目的集群，目的集群读不到，才会尝试原Redis单点；
	- 一旦进入该阶段，无法回滚；
* 写目的集群阶段
    - 此时，读写均发生在目的集群。可以停止后台的同步工具；
	- 清理业务里无用的sleep代码

经过以上5个阶段的处理，分布式锁就迁移到了新的集群。一旦经历了`拦截阶段`，很难回滚到之前的状态（两个Redis之间数据不一致）。因此该方案需要现场观察迁移情况，以防出现抖动，造成迁移失败。
	
## AOF rewrite导致Swap高占用
### 故障描述
某日，线上一Redis集群Swap飚高，查看日志，发现此时Redis正在进行AOF rewrite。AOF rewrite和Swap飚高有联系么？进一步查看日志发现，Swap飚高时，rewrite比正常情况下，多申请了很多`AOF buffer`，并且最终写入的数据也差了好几个数量级：
```
// Redis日志

// 正常情况
541554:C 28 Oct 21:13:23.859 * Concatenating 124.79 MB of AOF diff received from parent. // 124.79MB的diff
541554:C 28 Oct 21:13:30.675 * AOF rewrite: 9182 MB of memory used by copy-on-write

// Swap飚高
144162:M 28 Dec 10:54:03.804 * Background AOF buffer size: 1879 MB
144162:M 28 Dec 11:01:35.910 # Background AOF buffer size: 1972 MB
144162:M 28 Dec 11:01:42.915 # Background AOF buffer size: 1974 MB
144162:M 28 Dec 11:01:48.287 # Background AOF buffer size: 1977 MB
144162:M 28 Dec 11:04:36.502 # Background AOF buffer size: 1977 MB
144162:M 28 Dec 11:04:41.828 # Background AOF buffer size: 1972 MB                          // 申请了很多buffer size
652336:C 28 Dec 12:03:49.858 * Concatenating 31423.93 MB of AOF diff received from parent.  // 31423.93MB的diff
652336:C 28 Dec 12:06:17.057 * AOF rewrite: 205276 MB of memory used by copy-on-write
```
可以看到，正常情况下，AOF diff仅百兆级别，但Swap飚高时，竟达到了31423.93MB！`宝贵的内存资源就是被AOF buffer占用了`，导致Swap飚高，Swap飚高，导致整个系统变慢，恶性循环！

### 故障分析
AOF buffer的作用是啥，为何会分配这么多。此时需要详细了解一下AOF rewrite的流程。AOF触发后，为了不阻塞主进程，会fork出子进程，由子进程来完成大部分的重写工作：
1. Fork出子进程；
2. 重写主流程；
    - 2a) 子进程扫描db里所有的数据，并转换成对应的Redis命令，写入临时文件中；
    - 2b) 主进程会将fork之后，新写入的命令，追加到server 主进程会将fork之后，新写入的命令，追加到server.aof_rewrite_buf中，即`AOF buffer`；
3. 子进程2a完成后，通知主进程。主进程开始将累积的、重写缓存中的内容append到临时文件末尾；
4. 主进程rename临时文件，替换已有的AOF文件；

AOF rewrite流程非常清晰，但在极端场景下，会遇到问题：
* 3和4，是阻塞操作，主线程无法处理正常的请求；
* 2a过程中，若Redis被大量写入，加上Redis本身内存过大，扫描时间长，会导致AOF buffer会占用非常多的内存空间；

和查看监控后发现，Swap飚高时，业务正在大量的写入Redis，是写入高峰期。当`写入高峰期和后台的AOF rewrite重叠时`，导致Swap飚高。

### 解决方案
AOF自动触发的控制参数有两个：
* auto-aof-rewrite-min-size 10G
* auto-aof-rewrite-percentage 100

意味着AOF rewrite的自动触发的时机为：AOF文件大于10G，且当前AOF文件大小和最后一次AOF重写后的大小之间的比率>=100%时，会触发AOF rewrite。

缓解该问题的解决方案为：
1. 监控实际的AOF增长情况，调整auto-aof-rewrite-min-size，确保AOF rewrite不要发生在业务写入的高峰期；
2. 在低峰期，人为触发AOF rewrite（bgrewriteaof），保证高峰期时，AOF rewrite不会被自动触发；
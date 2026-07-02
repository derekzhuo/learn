# 一、CPU亲和性（CPU Affinity）解决什么问题

## 1. 核心定义

CPU亲和性：**把进程/线程绑定到一个或固定几个CPU核心上运行，不让操作系统随意在多个核心之间调度迁移**。

## 2. 要解决的4类核心问题（尤其你做仿真、DPIC、Daemon、Coemu这种低延迟高频通信场景最关键）

### （1）解决CPU跨核心调度带来的**Cache失效（最主要痛点）**

每个CPU核心都有独立L1/L2 Cache，多个核心共享L3。

- 线程在CPU0运行时，热点数据、代码会缓存到CPU0的L1/L2；
- OS如果把线程调度到CPU1，CPU1没有这份缓存，必须重新从内存加载，出现**Cache Miss**，性能暴跌、延迟抖动。
- 绑定CPU后，线程只在固定核心跑，Cache命中率大幅提升，延迟稳定。

### （2）解决进程频繁上下文切换、调度抖动

操作系统默认调度策略：哪个核心空闲就扔去哪，高频业务（仿真Daemon、DMA报文收发、MCLK时钟事件处理）频繁跨核会带来：

- TLB刷新、Cache失效、流水线清空；
- 调度延迟不可预测，出现毛刺、时序不准（你DPIC replay严格依赖MCLK时序，一旦调度抖动会导致激励下发时序错乱）。

### （3）解决多线程之间数据频繁跨NUMA节点访问

服务器一般是NUMA架构：不同CPU插槽的核心访问本地内存快，跨插槽内存访问延迟高。
把生产者、消费者线程绑定到**同一个NUMA节点的CPU核心**，可以避免跨NUMA内存访问，大幅降低延迟。

### （4）隔离业务，避免被其他进程抢占干扰

仿真、实时报文处理这类任务不能被后台日志、系统服务、其他业务进程抢占CPU。
通过CPU亲和性 + CPU隔离（内核`isolcpus`），把一组核心专门留给Daemon/Coemu进程，操作系统不会把其他进程调度到这些核心，保证时序确定性。

## 3. 一句话总结作用

> 
> CPU亲和性本质：**减少CPU上下文迁移、提升缓存命中率、降低调度抖动、保证业务时延确定性**，适合EDA仿真、网络收发包、实时控制、高频IPC/DMA报文交互这类对延迟敏感的场景。

# 二、CPU亲和性具体怎么做（Linux为主，最常用）

## 前置知识：CPU掩码

用**位掩码**表示绑定哪些核心：

- CPU0：`0b0001` → 十六进制 `0x01`
- CPU0+CPU1：`0b0011` → `0x03`
- CPU2：`0b0100` → `0x04`

## 方式1：命令行临时设置（taskset，最常用）

### 1）启动进程时直接绑定核心

```
# 将进程绑定到CPU2核心启动
taskset -c 2 ./dpic_daemon

# 绑定CPU 4、5、6三个核心
taskset -c 4,5,6 ./coemu_ip_sim

# 掩码方式绑定CPU0、1
taskset 0x03 ./testbench
```

### 2）给已经运行的进程设置亲和性

```
# PID=1234 绑定到CPU3
taskset -cp 3 1234

# 查看进程当前CPU亲和性
taskset -p 1234
```

## 方式2：C代码编程设置（sched\_setaffinity，服务常驻进程推荐）

```
#define _GNU_SOURCE
#include <sched.h>
#include <stdio.h>

// 绑定当前线程到CPU 2
int set_cpu_affinity(int cpu_id)
{
    cpu_set_t mask;
    CPU_ZERO(&mask);
    CPU_SET(cpu_id, &mask);
    return sched_setaffinity(0, sizeof(mask), &mask);
}

int main()
{
    set_cpu_affinity(2);
    while(1) {
        // Daemon高频轮询DMA报文、MCLK事件处理
    }
    return 0;
}
```

- `pid=0` 代表当前线程；
- 多线程场景：每个工作线程单独绑定不同核心，避免线程竞争同一CPU。

## 方式3：Shell脚本批量绑定 + 高级优化（工业仿真常用最佳实践）

### 步骤1：内核启动参数隔离CPU（可选，极致低延迟必备）

在grub内核参数添加：

```
isolcpus=2,3,4,5 nohz_full=2,3,4,5 rcu_nocbs=2,3,4,5
```

- `isolcpus`：操作系统不会把普通进程调度到这些核心；
- `nohz_full`：关闭这些核心的时钟滴答中断，减少调度干扰；
专门留给你的 Daemon、Coemu、TB 实时任务。

### 步骤2：线程架构绑定规范（适配你DPIC场景）

1. Daemon主线程：绑定**隔离大核CPU2**，处理DMA Channel2报文、Record缓存、MCLK时序匹配；
2. Coemu仿真线程：绑定同NUMA下CPU3，避免跨NUMA；
3. 日志、监控后台线程绑定非隔离核心（CPU0/1），不抢占业务核心。

## 方式4：多线程绑核避坑规则

1. 同一个CPU核心不要绑定多个高负载实时线程，会造成CPU争抢、上下文频繁切换；
2. 有生产者-消费者关系的两个线程，尽量绑定**同一NUMA节点不同核心**；
3. 不要跨NUMA随意绑核，会引入远程内存访问延迟；
4. 实时任务建议配合**SCHED\_FIFO/SCHED\_RR实时调度策略 + CPU亲和性**一起使用：

```
chrt -f 99 taskset -c 2 ./dpic_daemon
```

# 三、结合你的DPIC Daemon场景举例说明为什么需要绑核

1. Daemon不停轮询共享内存、Channel2 DMA报文收发、MCLK时间戳匹配、文件落盘；
2. 如果操作系统频繁把Daemon在多个核心来回调度：
   - DMA缓冲区、共享内存数据频繁Cache失效；
   - MCLK事件回调出现调度延迟，Type128/Type132报文时序错位；
   - Replay时`goto cycle_n`时序不准，激励下发延迟抖动，仿真复现bug失败；
3. 绑核后：线程固定在单一核心运行，缓存稳定、调度延迟可预测，保证DPIC录制/回放时序精准。

# 四、常见查看命令

```
# 查看CPU核心、NUMA节点
lscpu

# 查看线程运行在哪个CPU
ps -mp 进程PID -o THREAD,tid,psr

# 查看CPU中断亲和（可把DMA中断也绑定到业务CPU，进一步优化）
cat /proc/irq/xx/smp_affinity
```

需要我给你一份适配DPIC Daemon的**绑核+实时调度+NUMA优化部署脚本模板**吗？
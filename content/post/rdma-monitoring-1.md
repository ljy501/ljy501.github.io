---
title: "从零搭建 RDMA 监控系统：eBPF 把内核“黑盒”打开给你看"
date: 2026-10-28
categories: ["项目日志"]
tags: ["RDMA", "eBPF", "SoftRoCE", "bpftrace"]
---

> 这是《RDMA 容器集群网络监控》系列的上篇。今天讲怎么在**零硬件成本**下搭出一套 SoftRoCE 环境，再用 eBPF 把 RDMA 内核路径"打开"——看清楚一次压测到底触发了哪些内核事件。下篇再聊如何把这些事件变成 Prometheus 指标和 Grafana 面板。

## 一、前言：RDMA 很强，但它是个"黑盒"

RDMA（远程直接内存访问）这些年火得不行——低延迟、高吞吐、低 CPU 开销，分布式存储、AI 训练、高性能计算全都离不开它。它最大的特点就是**绕过内核**，让数据直接在网卡和应用内存之间流动。

但问题恰恰出在这个"绕过内核"上：**当 RDMA 跑在容器里，你几乎看不到它内部发生了什么**。

CPU、内存、网卡吞吐这些宏观指标，常规监控就能看；可 RDMA 软件协议栈里的 CQ（完成队列）、AH（地址句柄）什么时候被创建？一次压测触发了哪些内核事件？——传统工具全都"看不见"。

我的目标就一句话：**把 RDMA 内核路径"打开"给人看**。这一篇，先从最底层做起。

## 二、环境与架构：一张图看懂全貌

| 组件 | 用途 | 说明 |
|---|---|---|
| Linux SoftRoCE | 软件 RDMA | `rdma_rxe` 模块创建 `rxe0`，零硬件成本 |
| perftest | 压测负载 | `ib_send_bw` / `ib_write_bw` / `ib_read_bw` |
| bpftrace (eBPF) | 内核采集 | kprobe 动态追踪，无侵入 |
| Prometheus / Grafana | 存储展示 | 下一篇的主角 |

完整系统是四层链路：**负载层 → 采集层 → 指标层 → 展示层**，这一篇覆盖前两层。

![RDMA 容器监控指标采集与可视化总体架构](/images/bishi-architecture.png)

*整体架构：业务负载 → SoftRoCE 内核事件 → eBPF 采集 → Exporter → Prometheus → Grafana*

## 三、Step 1：零硬件搭建 SoftRoCE 环境

没有 InfiniBand 专用网卡怎么办？Linux 内核自带 **SoftRoCE**——在普通以太网上用软件模拟 RDMA 设备：

```bash
# 加载内核模块
modprobe rdma_rxe

# 基于宿主网卡创建软件 RDMA 设备 rxe0
rdma link add rxe0 type rxe netdev <你的网卡>

# 验证设备是否可用
ibv_devices
ibv_devinfo -d rxe0
```

SoftRoCE 的软件架构分四层，理解它有助于后面找追踪点：

![SoftRoCE 软件架构层次图](/images/bishi-softroce-arch.png)

*应用层（perftest）→ 用户态 Verbs（libibverbs）→ 内核 RDMA 子系统（rxe 驱动）→ 以太网设备层*

看到链路类型 `Ethernet`、传输类型 `IB`、端口 `ACTIVE`，说明设备已经映射进用户态工具链，可以开始压测了：

```bash
# 服务端先起来等待连接，客户端发起压测
ib_send_bw -d rxe0 127.0.0.1 --duration 4
```

## 四、Step 2：用 eBPF 盯住内核事件

环境就绪，现在做最"爽"的一步：**不重编译内核、不改一行内核代码**，用 bpftrace 的 kprobe 直接挂到 SoftRoCE 协议栈的函数上：

```bash
# 追踪完成队列（CQ）创建事件
bpftrace -e 'kprobe:rxe_create_cq { @[comm] = count(); }'

# 追踪地址句柄（AH）创建事件
bpftrace -e 'kprobe:rxe_create_ah { @[comm] = count(); }'
```

跑一次 `ib_send_bw` 压测，再看 bpftrace 的输出——事件计数就出来了。这一刻我**第一次"亲眼"看到 RDMA 通信在内核里到底触发了什么**：队列创建、地址句柄分配，全部一目了然。

采集过程完全**无侵入**：不改内核、不重编译、不影响业务流量，这就是 eBPF 最迷人的地方。

## 五、踩坑实录：这两个坑，我替你踩过了

**坑 1：内核模块没加载，设备起不来**

`rdma link add` 报错？先检查模块真的 load 了没：

```bash
lsmod | grep rdma_rxe
```

很多环境装完 rdma-core 就以为万事大吉，结果 `rdma_rxe` 压根没加载——这类"环境类问题"占了调试时间的一半以上。先把模块加载进系统，再谈建设备。

**坑 2：bpftrace 追踪点没触发**

脚本挂上了，压测也跑了，但计数永远是 0。常见原因是**追踪的函数名不对**：SoftRoCE 的符号是 `rxe_create_cq` 这类 `rxe_` 前缀，不是通用的 `ib_` 前缀。先确认内核里真实的符号名（`grep rxe_create /proc/kallsyms`），再挂 kprobe——函数名差一个字符，结果差十万八千里。

## 六、小结与预告

这一篇完成了两件事：

1. **零硬件跑通 RDMA**：SoftRoCE + `rxe0`，perftest 压测正常
2. **eBPF 打开黑盒**：bpftrace 捕获 `rxe_create_cq` / `rxe_create_ah` 内核事件，无侵入、不重编译

但这些还只是"能看"，不是"能监控"——事件计数还不能存储、查询、展示。

**下篇预告**：《从内核事件到仪表盘：Prometheus + Grafana 打造 RDMA 可观测链路》——我会把事件计数变成 Prometheus 指标、接入 Grafana 面板，再把整套链路塞进 Docker 容器验证一遍，包括那个让我差点放弃的 memlock 坑。

---

*本系列整理自毕业设计论文《面向 RDMA 的容器集群网络监控与可视化系统的设计与实现》，数据均为论文实测结果。*

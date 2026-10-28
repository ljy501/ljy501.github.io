---
title: "从内核事件到仪表盘：Prometheus + Grafana 打造 RDMA 可观测链路"
date: 2026-10-28
categories: ["项目日志"]
tags: ["RDMA", "Prometheus", "Grafana", "Docker", "eBPF"]
---

> 这是《RDMA 容器集群网络监控》系列的下篇。上一篇我用 SoftRoCE + bpftrace 把 RDMA 内核事件"抓"了出来，但这还只是"能看"，不是"能监控"。今天把它变成真正的监控系统：**Python Exporter 导出指标 → Prometheus 存储查询 → Grafana 可视化 → Docker 容器化验证**。

## 一、从"事件"到"指标"：还差一个 Exporter

bpftrace 只能看，不能存。要让数据进监控栈，得先把事件计数转成 **Prometheus 文本格式指标**。

我写了一个 Python exporter：定时拉取 bpftrace 的事件计数，整理成指标暴露在 `9109` 端口的 `/metrics` 接口上：

```bash
curl http://127.0.0.1:9109/metrics

# rdma_rxe_create_cq_total 16
# rdma_rxe_create_ah_total 41
```

数据流从这一步开始形成完整闭环：

![系统数据流图](/images/bishi-datastream.png)

*监控数据从 RDMA 事件产生开始，经过采集、指标化、存储、展示，形成闭环*

## 二、Prometheus 接入：三行配置搞定

Prometheus 那边加一个抓取任务：

```yaml
scrape_configs:
  - job_name: rdma-ebpf
    static_configs:
      - targets: ["127.0.0.1:9109"]
```

验证两步走：

```bash
# 健康检查
curl http://127.0.0.1:9090/-/healthy
# Prometheus Server is Healthy

# PromQL 查询指标
curl 'http://127.0.0.1:9090/api/v1/query?query=rdma_rxe_create_cq_total'
```

看到抓取目标 `up`、查询能返回指标值，存储查询链路就通了。

## 三、Grafana 面板：最朴素的 PromQL 就够用

数据进了 Prometheus，可视化水到渠成。仪表盘核心就三个面板，全是最基础的 PromQL：

| Grafana 面板 | PromQL | 说明 |
|---|---|---|
| CQ 总次数 | `rdma_rxe_create_cq_total` | 累计值 |
| AH 总次数 | `rdma_rxe_create_ah_total` | 累计值 |
| 近 1 分钟事件增量 | `increase(rdma_rxe_create_cq_total[1m])` | 增量趋势 |

![Grafana RDMA SoftRoCE Monitor 仪表盘](/images/bishi-grafana.png)

*仪表盘实测：CQ 累计 35.8、AH 累计 41.2，增量曲线实时滚动*

## 四、容器化验证：把压测负载塞进 Docker

光在宿主机上跑通不算数，RDMA 在真实场景里是跑在容器里的。我构建了一个容器化 RDMA 测试负载：

```bash
docker run --rm --network host --privileged \
  --ulimit memlock=-1:-1 --cap-add IPC_LOCK \
  -v /dev/infiniband:/dev/infiniband \
  -v /sys:/sys:ro \
  rdma-perftest:22.04 \
  ib_send_bw -d rxe0 127.0.0.1 --duration 4
```

验证流程是标准五步：环境准备 → 负载触发 → 指标采集 → 查询验证 → 可视化展示。

![RDMA 监控实验验证流程](/images/bishi-test-flow.png)

## 五、踩坑实录：容器化适配，差点把我劝退

**坑 1：容器里找不到 RDMA 设备**

`ibv_devices` 在容器里输出为空，第一反应以为是镜像缺依赖。其实真正原因是：**容器默认看不到宿主机的 `/dev/infiniband`**。解法就在上面的 docker run 参数里：

- `--network host`：RDMA 走主机网络栈
- `-v /dev/infiniband:/dev/infiniband`：把 RDMA 设备文件挂进容器
- `-v /sys:/sys:ro`：内核相关 sysfs 只读挂载

**坑 2：`ib_send_bw` 报 mlock 相关错误**

RDMA 需要锁定内存，容器默认的 memlock 限制会让压测直接失败。`--ulimit memlock=-1:-1 --cap-add IPC_LOCK` 就是干这个的：**先调 ulimit，再补 capabilities**，顺序反了照样报错。

## 六、测试结果：数据说话

### 带宽测试

perftest 三件套 × 4 种消息大小（64B ~ 65536B），RC 连接：

| 消息大小 | Write 带宽 (MB/s) | Read 带宽 (MB/s) | Send 带宽 (MB/s) |
|---|---|---|---|
| 64B | 28.21 | 16.23 | 26.06 |
| 1024B | 391.57 | 241.15 | 354.94 |
| 4096B | 768.34 | 561.98 | 704.26 |
| 65536B | **1080.57** | 710.26 | **1036.89** |

结论：**消息越大带宽越高**（固定开销被摊薄），Write 和 Send 性能接近，Read 因软件路径开销偏弱，SoftRoCE 环境下的正常现象。

### eBPF 采集有效性（三轮压测）

| 轮次 | 压测前 | 压测后 | 增量 | Prometheus 查询 | 平均带宽 |
|---|---|---|---|---|---|
| 1 | 4 | 8 | +4 | 8 | 1106.49 MB/s |
| 2 | 8 | 12 | +4 | 12 | 1100.98 MB/s |
| 3 | 12 | 16 | +4 | 16 | 1103.99 MB/s |

三轮增量稳定、Prometheus 与 exporter 完全一致：**eBPF 真的"看见"了 RDMA 通信，不是纸面指标**。

### 容器化验证（最关键）

| 测试项目 | 结果 |
|---|---|
| 压测前 `rdma_rxe_create_cq_total` | 16 |
| 压测后 | **20**（增量 +4） |
| Prometheus 查询值 | 20（与 exporter 一致） |
| 平均带宽 | **1107.50 MB/sec** |
| 客户端返回码 | 0 |

![Docker 容器化 RDMA 负载监控验证结果](/images/bishi-docker-result.png)

**容器里的 RDMA 压测，成功触发了宿主机的内核事件，并被整条 eBPF → Prometheus → Grafana 链路完整观测到**，这套方案在真实容器化场景下是可行的。

## 七、总结与思考

做完这个项目，最深的体会有三点：

1. **"可观测性"的价值是打开黑盒**。以前监控就是"看 CPU 满了没"，这套链路看到的是**协议栈内部每一步发生了什么**，这个思维转变比技术本身更值钱。
2. **三段链路，每一环都得口径一致**。事件计数 → 指标模型 → 面板展示，任何一环对不上，"监控了个寂寞"。
3. **踩坑的过程就是成长**。memlock、设备挂载、capabilities，每一个坑都让我对 Linux 权限模型和内核设备管理理解更深。

这套 **eBPF → Prometheus → Grafana** 方法论完全可复用：换一个内核函数，就能监控另一种内核行为。后续我计划往 K8s 方向扩展，把节点级、容器级、集群级的监控维度都补上。

---

*本系列整理自毕业设计论文《面向 RDMA 的容器集群网络监控与可视化系统的设计与实现》，数据均为论文实测结果。上篇见 [从零搭建 RDMA 监控系统：eBPF 把内核"黑盒"打开给你看](/post/rdma-monitoring-1/)。*

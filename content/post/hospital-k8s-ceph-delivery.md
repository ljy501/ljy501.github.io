---
title: "给医院机房交付 7 台服务器：K8s 高可用与 Ceph 分布式存储落地实录"
date: 2026-10-28
categories: ["项目日志"]
tags: ["Kubernetes", "Ceph", "HAProxy", "Keepalived", "服务器交付"]
---

## 一、前言

这个项目是给深圳某医院的信息通信与算力基础设施项目交付算力底座，简单说就是把 7 台服务器在 15 天内从拆箱变成一套能跑的容器化平台。硬件不是全新一批，里面有 5 台新采购的 Dell PowerEdge，还有 2 台利旧服务器，CPU 型号、内存规格、磁盘形态全不一样。要把它们凑成一个 K8s 集群加一套分布式存储，还要保证高可用，这个混搭方案本身就是最大的坑。

整篇按交付流程来写，重点放在高可用控制平面和 Ceph 存储这两块，数据全部来自交付文档。

## 二、环境与架构

7 台服务器的角色分配如下。

| 节点 | 型号 | CPU | 内存 | 磁盘 | 角色 |
|---|---|---|---|---|---|
| 节点 1 | Dell R760 | 2×Xeon Gold 6430 | 256GB DDR5 | 2×1.92TB SSD + 2×7.68TB NVMe | K8s Worker |
| 节点 2 | Dell R760 | 2×Xeon Gold 6430 | 256GB DDR5 | 2×1.92TB SSD + 1×7.68TB NVMe | K8s Worker |
| 节点 3 | Dell R760 | 2×Xeon Gold 6430 | 256GB DDR5 | 2×1.92TB SSD | K8s Worker |
| 节点 4 | Dell R760xs | 2×Xeon 4510 | 128GB DDR5 | 2×960GB SSD + 1×20TB SAS HDD | K8s Master + Ceph Mon + OSD |
| 节点 5 | Dell R760 | 2×Xeon Gold 6430 | 256GB DDR5 | 2×1.92TB SSD + 1×7.68TB NVMe | 应用部署 + CephFS 挂载 |
| 节点 6 | Dell R740XD（利旧） | 2×Xeon 6248R | 384GB | SATA SSD RAID1 + 12TB/20TB SATA RAID5 | K8s Master + Ceph Mon + OSD |
| 节点 7 | Dell R740XD（利旧） | 2×Xeon 6248R | 256GB | SATA SSD RAID1 + 12TB/20TB SATA RAID5 | K8s Master + Ceph Mon + OSD |

控制面是三 Master 高可用：节点 4、6、7 跑 etcd，HAProxy 加 Keepalived 对外提供 VIP。Worker 是节点 1、2、3，节点 5 承担应用部署和 CephFS 挂载。

网络侧是三网分离：管理网走 iDRAC 专用 1GbE，业务网（eth0）承载 K8s 控制面和 Pod 流量，存储网（eth1）专给 Ceph。交换设备是一台华为 S6730S-S24X6Q-A 万兆交换机，配了 2 块 25Gb 双端口网卡和 20 个万兆多模光模块。业务和存储端口都配 Trunk，存储网 VLAN 200 单独配了 QoS 保障带宽，这个后面踩坑部分细说。

![Dell PowerEdge R760 服务器](/images/hospital_p2_img1.jpeg)

## 三、核心实现

### 1. 利旧服务器预处理

节点 6、7 是旧的 R740XD，不能直接拿来用。先做全量数据备份，用 dd 或者 Veeam，备份完要出 MD5 加 SHA256 校验报告。然后把原有 RAID 配置清掉，阵列卡切到 HBA 模式，让磁盘以直通方式暴露给系统。验证命令是 `lsscsi -g`，输出里能看到 disk 类型而不是 raid 才算过。

这一步不做，后面 Ceph 的 OSD 初始化会拿不到整块盘。

### 2. 系统基础配置

所有节点统一装 Ubuntu 22.04 LTS Minimal。装完先关掉透明大页和 Swap，K8s 和 Ceph 都要求这两个不能开。

```bash
echo 'never' > /sys/kernel/mm/transparent_hugepage/enabled
echo 'never' > /sys/kernel/mm/transparent_hugepage/defrag
swapoff -a
sed -i '/ swap / s/^/#/' /etc/fstab
```

内核参数按生产标准调，主要给高负载和故障恢复兜底。

```bash
cat <<EOF | sudo tee -a /etc/sysctl.conf
vm.swappiness=1
net.ipv4.ip_nonlocal_bind=1
net.ipv4.tcp_keepalive_time=600
net.ipv4.tcp_retries2=5
vm.overcommit_memory=1
kernel.panic=10
kernel.panic_on_oops=1
EOF
sysctl -p
```

防火墙默认策略 INPUT 直接 DROP，只放行必要端口。SSH 限定管理 IP，K8s 组件端口（6443、2379、2380、10250 等）按需开放，Ceph 的 Mon 6789 和 OSD 6800-7300 也要放。这里容易漏的是 30025 端口，节点 4、6、7 上跑着 conda 和 R 环境，远程访问要用它，忘了开会导致应用侧排查半天。

### 3. 高可用控制平面

etcd 集群跑在节点 4、6、7 上，数据目录挂到性能最好的本地 SSD，启用 TLS 双向认证。关键配置如下。

```yaml
name: etcd-${HOSTNAME}
data-dir: /var/lib/etcd
listen-client-urls: https://${NODE_IP}:2379
listen-peer-urls: https://${NODE_IP}:2380
initial-cluster:
  etcd-node1=https://<Node1_IP>:2380,etcd-node6=https://<Node6_IP>:2380,etcd-node7=https://<Node7_IP>:2380
initial-cluster-token: my-k8s-etcd
initial-cluster-state: new
client-cert-auth: true
trusted-ca-file: /etc/etcd/ca.crt
auto-compaction-retention: "1000"
```

API Server 的入口是 HAProxy 加 Keepalived 的组合。HAProxy 在所有 Master 上跑，配置 6443 的 TCP 代理，后端是三个 Master，开启 TCP 健康检查。Keepalived 负责 VIP 漂移，主节点优先级设最高，备节点低一档，用脚本检查 haproxy 进程存活，挂了就触发切换。

```haproxy
frontend k8s-api
    bind VIP:6443
    mode tcp
    option tcplog
    default_backend k8s-masters

backend k8s-masters
    mode tcp
    balance roundrobin
    option tcp-check
    server master1 <Node1_IP>:6443 check check-ssl verify none inter 10s rise 2 fall 2
    server master6 <Node6_IP>:6443 check check-ssl verify none inter 10s rise 2 fall 2
    server master7 <Node7_IP>:6443 check check-ssl verify none inter 10s rise 2 fall 2
```

集群初始化用 kubeadm，指定 `--control-plane-endpoint=<VIP>:6443`，这样所有组件都走 VIP，Master 挂一台不影响 API 可用性。之后把节点 6、7 作为额外控制面加入，Worker 节点 1、2、3、5 正常 join，网络插件选的 Calico。

### 4. Ceph 分布式存储

Monitor 部署在节点 4、5、7，三个奇数节点保证选举。Mon 数据放可靠性高的 SSD 上。OSD 初始化按磁盘形态分配，NVMe 直接做高性能 OSD，SATA SSD 拆成两个 OSD，20TB 的 SAS HDD 做高容量 OSD。

```bash
ceph-volume lvm create --data /dev/nvme0n1   # NVMe 高性能 OSD
ceph-volume lvm create --data /dev/sdb       # SATA SSD OSD
```

容易忽略的是 Journal 分区。SATA SSD OSD 和 HDD OSD 强烈建议在 SSD 上划出专用分区当 Journal，每个 OSD 至少 5GB，不然写性能会明显拖垮。CRUSH Map 按 Root 到 Rack 到 Host 的层次设计，让数据均匀落在不同物理机和磁盘类型上。存储池按用途分，K8s 用 RBD，应用侧用 CephFS。CephFS 挂载到应用节点时，fstab 里必须带 `_netdev` 选项，不然开机顺序不对会挂载失败。

```
<CephFS_Mon_IP>:/ /mnt/cephfs fuse.ceph ceph.name=admin,ceph.conf=/etc/ceph/ceph.conf,_netdev,noatime 0 0
```

## 四、踩坑记录

**利旧服务器规格差异**。节点 6、7 和新买的 R760 在 CPU、内存、磁盘性能上差一大截，尤其 64GB 内存档位偏紧。K8s 组件版本必须统一，Master 和 Mon 都压在利旧节点上时，资源使用要盯紧。方案里直接建议了对这两个节点做资源监控，不然跑着跑着 OOM 排查成本很高。

**Journal 分区没配够**。文档里明确要求每个 OSD 至少 5GB 的专用 SSD Journal，HDD 和 SATA SSD OSD 必须配。第一次规划时容易把这步当成可选项，实际跑起来 Ceph 的写延迟会明显恶化，这个属于不配就返工的项。

**存储网络 QoS 是硬指标**。交换机上要给存储网 VLAN 200 配置不低于 8Gbps 的保障带宽。存储流量和业务流量混在一个交换机上，没有 QoS 的话一次大规模回写就能把网络打满，直接影响集群稳定性。这个在方案里被标成关键项，不是建议。

**单节点角色过载**。节点 7 同时承担 Master、etcd、Ceph Monitor 三个角色，负载压力集中。方案的处理是明确要求密切监控，Mon 数据必须放在可靠介质上，避免把控制面节点当成纯存储节点用。

## 五、测试与交付

整个实施按 15 天排期推进。

| 时间 | 内容 |
|---|---|
| Day 1-2 | 物理上架、连线、标签；利旧服务器备份与 HBA 模式验证 |
| Day 3-4 | 全节点 OS 安装与基础配置 |
| Day 5-7 | etcd、HAProxy、Keepalived 部署，K8s 控制面初始化，Worker 加入 |
| Day 8-10 | Ceph Monitor 部署，OSD 初始化与 CRUSH Map，CephFS 挂载 |
| Day 11-13 | 系统集成测试、压力测试、HA 切换与故障模拟 |
| Day 14-15 | 验收测试、问题修复、文档整理与交付 |

集成测试重点验三块：VIP 切换时 API 是否持续可用，节点故障时 K8s 能否自愈，Ceph 数据读写是否稳定。这三块过了，才算真正交付。

## 六、总结

这个项目做完，最大的体会是混搭硬件做容器化平台，坑基本集中在两类。一类是硬件层，利旧设备必须先做数据备份和 RAID 直通，规格差异要靠监控兜底。另一类是配置层，Journal 分区、QoS 保障、防火墙端口这些看着不起眼的项，漏一个都会在集成测试阶段爆出来。

高可用方案本身是成熟组合：etcd 保数据一致性，HAProxy 保入口可达，Keepalived 保 VIP 漂移，Ceph 保存储冗余。把这套在真实机房完整走一遍，比看十遍文档有用得多。

> **Abstract**：This project delivered a containerized infrastructure for a hospital, turning 7 heterogeneous servers (5 new Dell PowerEdge and 2 reused R740XD) into a high-availability Kubernetes cluster with Ceph distributed storage within 15 days. Key work included reused-server RAID/HBA preprocessing, a 3-master control plane with etcd, HAProxy and Keepalived VIP, and Ceph OSD/Journal/CRUSH planning. The article also records practical pitfalls such as hardware heterogeneity, Journal partition requirements and storage network QoS.

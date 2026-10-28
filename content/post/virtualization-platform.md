---
title: "给 7×24 业务搭一套 ESXi 虚拟化平台：vMotion 不停机维护，主机挂了分钟级自愈"
date: 2026-10-28
categories: ["项目日志"]
tags: ["VMware vSphere", "ESXi", "vMotion", "高可用", "虚拟化"]
---

> I built a VMware vSphere virtualization platform from scratch for a 7×24 production-style application: two ESXi hosts managed by vCenter, shared iSCSI storage, and three Ubuntu VMs (Web / Application / Database). This article walks through the architecture, the build sequence, and how vMotion plus HA delivered no-downtime hardware maintenance and minute-level auto recovery after host failure.

## 一、前言：一台物理机宕机，全站陪它陪葬

传统部署有个让人睡不着觉的问题：**应用和硬件绑死了**。Web 服务器装在 A 机器上，你想给 A 机器清灰、加内存、换电源？先通知业务方停机窗口。更糟的是 A 机器半夜主板烧了，应用跟着一起挂，Recovery 全靠人工现场处理。

这次课设的目标很硬核：给一个 7×24 对外服务的应用系统（我拿个人博客当被测对象）搭一套虚拟化平台，硬性指标三条：

1. **不停业务**就能对服务器硬件做维护/升级；
2. 主机硬件故障后，应用**分钟级自动重启**恢复服务；
3. Web Server 和 Application Server 必须落在**不同物理主机**，Application 和 Database 落在**同一台**。

这三条，恰好对应 vSphere 的三件套：**vMotion、HA、DRS/手动主机策略**。

## 二、环境与架构：一张图看懂全貌

| 组件 | 用途 | 说明 |
|---|---|---|
| VMware Workstation | 桌面虚拟化 | 在本机上"虚拟"出整个数据中心的运行环境 |
| ESXi 5.5 ×2 | 裸机 Hypervisor | exsi1.lab.net、exsi2.lab.net，真正跑 VM 的宿主 |
| Windows Server 2008 R2 + vCenter 5.5 | 集中管理 | 建数据中心、加主机、配集群，Web Client 操作 |
| iSCSI 共享存储 | 虚拟机磁盘 | VMFS 数据存储挂到两台主机，vMotion/HA 的前提 |
| Ubuntu 24.04 LTS ×3 | 业务虚拟机 | Web(192.168.74.128) / App(192.168.74.129) / DB(192.168.74.130) |
| 宝塔面板 | 环境与部署 | 一键装 Nginx / Tomcat / MySQL，前后端分离部署 |

![vSphere 虚拟化平台总体架构：管理员经 vSphere Client 管理两台 ESXi 主机，其上运行三台业务虚拟机](/images/virtualization-architecture.png)

*管理员笔记本 → vSphere Client / Web Client → Esxi1、Esxi2 两台主机（挂 iSCSI 共享存储）→ Web / Application / Database 三台虚拟机对外提供服务*

一个关键设计决策：**共享存储是 vMotion 和 HA 的地基**。虚拟机磁盘必须放在两台主机都能访问的 iSCSI 数据存储上，迁移时才不用搬盘，主机挂了另一台才能直接拉起 VM。

## 三、Step 1：搭 ESXi 双主机 + vCenter 管控面

搭建顺序是层层套娃：

```text
Workstation
├── ESXi 主机 1（exsi1.lab.net）
├── ESXi 主机 2（exsi2.lab.net）
└── Windows Server 2008 R2（LB_VC.lab.net / 192.168.74.147）
     └── vCenter Server 5.5
          ├── 创建 Datacenter
          ├── 添加 exsi1 / exsi2 两台主机
          └── 建集群，开启 vMotion 与 HA
```

网络上先把 lab.net 内网跑通：vCenter 主机名注册进 DNS，简单安装向导里主机名填 `LB_VC.lab.net`（192.168.74.147），先决条件检查全部打绿，主机名、FQDN、IP、域加入、DNS 解析缺一不可，任何一项红叉都会卡住安装。Web Client 用 `https://<vcenter>:9443/vsphere-client` 登录。添加主机时前提检查列表全绿（DNS 解析成功、域加入正常），主机才会出现在清单里。

这一步的经验是：**vSphere 的坑八成先出在网络和命名上**。两台 ESXi 主机、vCenter、共享存储之间必须用主机名互通，而不是 IP，迁移、HA 心跳都靠主机名建信任关系。实验环境里我专门先搭了一台 Windows Server 2008 R2 当 DNS 服务器，把 exsi1.lab.net、exsi2.lab.net 都解析好，后面添加主机、挂载存储、配 vMotion 才一路绿灯。

## 四、Step 2：共享存储与三台业务虚拟机

存储这一侧，两台 ESXi 主机都挂上同一个 iSCSI Target，格式化为 VMFS 数据存储。这一步做完，后面所有虚拟机的磁盘都落在共享盘上：**没有它，vMotion 直接报错**。

业务侧以 Web Server（Ubuntu-24.04-live-server 镜像）为模板，先把它转成模板再克隆批量产出 Application Server 和 Database Server，避免三台机器手工装三遍系统。三台 VM 起来后，分别装宝塔面板：

```bash
# Ubuntu 24.04 上安装宝塔面板（官方一键脚本）
wget -O install.sh https://download.bt.cn/install/install-ubuntu_6.0.sh
sudo bash install.sh
```

进面板后推荐安装套件选 JAVA 环境（Nginx 1.26 + MySQL 5.7 + JDK 1.8），和 LNMP/LAMP 并列：

![宝塔面板推荐安装套件：LNMP / LAMP / JAVA 环境一键部署](/images/virtualization-bt-panel-lnmp.png)

*三台 VM 分工：Web 机跑 Nginx 承接前端静态页面，App 机跑 Tomcat 后端接口，DB 机跑 MySQL*

## 五、Step 3：前后端分离部署

代码按 IP 严格分机：**前端静态页面丢到 Web Server（.128），后端部署到 Application Server（.129），数据库建在 Database Server（.130）**。Nginx 把 API 请求反代到 .129，后端再连 .130 的 MySQL，这就是经典的三层分离拓扑，和前面"Web/App 不同主机、App/DB 同主机"的要求一一对应。

部署完浏览器访问 `http://192.168.74.128`，博客首页正常出来，三台虚拟机正式上岗：

![部署完成后的个人博客首页，跑在 Web Server 192.168.74.128 上](/images/virtualization-blog-running.png)

*应用系统 A 上线：首页、项目集、关于我全部可访问，前后端分离调用链路打通*

## 六、Step 4：两项高可用能力的验证

**验证 1：vMotion 不停机维护**

要维护 exsi1 时，在 Web Client 上对它上面的虚拟机右键"迁移"→ 更改主机，vMotion 实时把运行中的 VM 从 exsi1 搬到 exsi2。迁移过程中刷新博客页面，**页面一直正常显示、功能照常可用**，用户完全无感知，维护窗口就这样消失了。

**验证 2：HA 分钟级故障自愈**

先把两台主机放进同一个集群，开启 vSphere HA：勾选"打开 vSphere HA"、启用主机监控、设置好接入控制：

![vSphere 集群设置：开启 HA、启用主机监控](/images/virtualization-ha-config.png)

然后**挂起 exsi1 模拟硬件宕机**。现象很真实：页面刷新开始转圈加载，加载失败，因为跑在 exsi1 上的应用被"掐"了。等了一两分钟，集群检测到主机失联，自动在 exsi2 上把虚拟机重启起来，页面**突然恢复正常**。从宕机到自愈，分钟级完成，全程不需要人到场。

## 七、踩坑实录

**坑 1：vMotion 迁移报"未满足兼容性要求"**
最常见原因就是**共享存储没挂对**：虚拟机磁盘还在主机本地 datastore 上，另一台主机够不着，迁移自然失败。排错顺序：先看两台主机是否都挂载了同一个 iSCSI VMFS，再看 VM 磁盘是否在共享存储上，然后是 vMotion 虚拟交换机网段。

**坑 2：主机挂起后 HA 没自动拉起**
现象：exsi1 挂了，VM 安安静静躺在那没人管。原因是**HA 的"主机监控"没勾上**，只开了 HA 开关而没启用主机监控，集群不会主动判定主机失联。另外 vCenter 与主机之间网络要通，管理网络不通，HA 心跳也会失效。

**坑 3：证书报错大红叉**
vSphere Web Client 用自签名证书，浏览器（当年还是 IE）满屏"证书错误"。实验环境里直接点继续访问即可，但生产环境这是安全隐患，正式环境必须替换企业 CA 签发的证书。

## 八、总结与思考

这套平台最终交付了什么：

| 指标 | 要求 | 实测结果 |
|---|---|---|
| 硬件维护不中断 | 业务零停机 | vMotion 迁移中页面全程正常 |
| 故障自愈 | 分钟级恢复 | 挂起 exsi1 后约 1–2 分钟自动恢复 |
| 主机分布 | Web/App 分主机，App/DB 同主机 | 手动部署落位，符合拓扑要求 |

回头看，虚拟化的本质不是"省机器"，而是**把硬件和应用解耦**：应用不再关心它跑在哪块 CPU 上，迁移、故障切换、资源调度才有了操作空间。当然这套实验环境也有明显短板，存储 I/O 性能、网络带宽的动态分配、智能监控告警都还很初级。真要演进，下一步就是接 SDN、上监控平台，再往混合云走。

---

*本文整理自课程设计《虚拟化服务器平台搭建》，环境 IP、验证过程与现象描述均为文档实测记录。*

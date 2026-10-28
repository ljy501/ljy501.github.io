---
title: "从单臂路由到 OSPF：VLAN、动态 NAT、ACL 防火墙一次配通的组网实战"
date: 2026-10-28
categories: ["项目日志"]
tags: ["网络工程", "VLAN", "OSPF", "NAT", "ACL"]
---

> A campus network must segment staff and dorm traffic, route between VLANs, reach the Internet through NAT, and enforce firewall rules: dorm users may surf the web and reach the teaching block, but the teaching block may only hit the data-center server's HTTP/FTP services. This lab builds the whole thing in Cisco Packet Tracer — VLAN trunking, router-on-a-stick inter-VLAN routing, a three-layer switch, OSPF dynamic routing, dynamic NAT with PAT, and both standard and extended ACLs — then verifies each policy with ping and browser tests.

## 一、前言：一张"要求互相矛盾"的网怎么配

实验需求听起来简单：一个学院，教职工分散在两栋教学楼，学生分散在两栋宿舍楼，外加一台数据中心服务器、一条连 ISP 的串口链路。但访问控制的要求是互相咬着的：

- 宿舍楼要能上外网，**教学楼不能上外网**；
- 宿舍楼能访问教学区，也能访问外网；
- 教学楼**只能**访问数据中心的 WWW 和 FTP，别的一概不行。

要同时满足这几条，VLAN 划分、VLAN 间路由、动态路由、NAT、ACL 一个都跑不掉。这篇把整条配置链拆开记一遍。

## 二、拓扑与编址：先把网图钉死

整网在 Packet Tracer 里长这样：核心是一台 2811（Router1），教学区挂一台 21XM 路由器（Router2），宿舍区挂一台 3560 三层交换机，数据中心和 ISP 各挂一边：

![实验拓扑图：核心 Router1 串起教学区 Router2、宿舍区三层交换机、数据中心 Server-PT 与外网 Router ISP，各 VLAN/接口 IP 全部标注](/images/netlab-topology.png)

关键编址表：

| 设备/网段 | 接口 | IP 地址 | 网关 |
|---|---|---|---|
| 核心 Router1 | Fa0/0 | 192.168.4.101/24 | — |
| 核心 Router1 | Fa0/1 | 172.16.1.101/24 | — |
| 核心 Router1 | Fa1/0 | 10.1.2.101/24 | — |
| 核心 Router1 | Se0/3/0 | 202.121.241.8/24 | — |
| Router ISP | Se0/0/0 | 202.121.241.100/24 | — |
| PC1（教学楼1）VLAN 2 | — | 192.168.2.22/24 | 192.168.2.254 |
| PC2（教学楼2）VLAN 3 | — | 192.168.3.22/24 | 192.168.3.254 |
| PC3（宿舍楼1）VLAN 4 | — | 172.16.4.22/24 | 172.16.4.254 |
| PC4（宿舍楼2）VLAN 5 | — | 172.16.5.22/24 | 172.16.5.254 |
| Server-PT（数据中心） | Fa0 | 10.1.3.22/24 | — |

一个铁律先记死：**和主机直连的三层接口/VLAN 虚接口 IP，必须和主机网关一致**，否则二层再通也 ping 不通，这个坑后面还会踩一次。

## 三、Step 1：分区配置，教学区走单臂路由，宿舍区走三层交换机

两个区域用了不同的 VLAN 间路由方案，正好对比着看。

**教学区用单臂路由**：二层交换机起 VLAN 2、VLAN 3 并把端口划进去，上联口设 trunk；路由器 Router2 的 Fa0/1 不开物理 IP，而是开两个子接口，用 802.1Q 打标签封装：

```text
Router2(config)# interface fa0/1.1
Router2(config-subif)# encapsulation dot1Q 2      ! 封装 VLAN 2
Router2(config-subif)# ip address 192.168.2.254 255.255.255.0
Router2(config-subif)# interface fa0/1.2
Router2(config-subif)# encapsulation dot1Q 3      ! 封装 VLAN 3
Router2(config-subif)# ip address 192.168.3.254 255.255.255.0
```

**宿舍区用三层交换机**：二层交换机起 VLAN 4/VLAN 5，三层交换机上给 SVI 配 IP，上联口设 trunk。三层接口要先 `no switchport` 才能配 IP：

```text
! 三层交换机上把物理口转为三层路由口后，当路由器用
Switch(config)# interface fa0/1
Switch(config-if)# no switchport
Switch(config-if)# ip address 172.16.1.254 255.255.255.0
```

分区内部先互 ping 通（PC3↔PC4、PC1↔PC2），再往下走。

## 四、Step 2：OSPF 把各区域拉成一张网

区域内通了还不够，区域之间、到数据中心、到 ISP 都得有路由。选 OSPF 而不是静态/RIP 的原因：**可扩展性比静态好配，又不像 RIP 那样 16 跳就判死**。在每个路由器和三层交换机上宣告各自直连网段：

![核心路由器 OSPF 配置 CLI：router ospf 1 后逐条 network 宣告网段，日志输出 OSPF-5-ADJCHG 邻居 FULL](/images/netlab-ospf-config.png)

```text
Router1(config)# router ospf 1
Router1(config-router)# network 172.16.1.0 0.0.0.255 area 0
Router1(config-router)# network 192.168.4.0 0.0.0.255 area 0
Router1(config-router)# network 10.1.2.0 0.0.0.255 area 0
Router1(config-router)# network 202.121.241.0 0.0.0.255 area 0
```

注意 OSPF 的 network 后面跟的是**反掩码（wildcard）**，不是子网掩码，`0.0.0.255` 而不是 `255.255.255.0`。配完看日志刷出 `%OSPF-5-ADJCHG: ... Neighbor ... on FastEthernet0/1 from LOADING to FULL`，邻居关系才真正建起来。特权模式下 `show ip route` 能看到 O 开头的路由，此时全网 PC 和 server 全部互通。

## 五、Step 3：动态 NAT 让宿舍区共享公网出口

要求宿舍区能上外网、教学楼不能。在核心路由器上做动态 NAT，地址池加 `overload` 做端口复用（PAT），让多个私网主机共享几个公网 IP：

```text
! 1) 用 ACL 圈定要转换的私网段
access-list 1 permit 172.16.0.0 0.0.255.255

! 2) 建公网地址池
ip nat pool mynet 114.114.114.111 114.114.114.118 netmask 255.255.255.0

! 3) ACL 绑到地址池，overload 开启端口复用
ip nat inside source list 1 pool mynet overload

! 4) 指明 inside / outside 接口
interface fa0/1
 ip nat inside
interface se0/3/0
 ip nat outside
```

`overload` 是灵魂：没有它，公网池里 8 个 IP 是一对一占坑，占满就得排队；开了 overload，一个公网 IP 靠不同端口号能同时塞下一堆私网会话，省地址。配完用 `show ip nat translations` 看转换表，`inside global` 是转换后的公网[IP:端口]，`inside local` 是转换前的私网[IP:端口]。

## 六、Step 4：ACL 防火墙，宿舍楼到数据中心一律拦死

先拦"宿舍楼不许碰数据中心"。在核心路由器连数据中心的 Fa1/0 出口方向挂标准 ACL：

```text
access-list 1 deny 172.16.0.0 0.0.255.255   ! 拒绝宿舍区
access-list 1 permit any                     ! 其余全放行（默认 deny any，这句必须有）
interface f1/0
 ip access-group 1 out
```

用宿舍区 PC3 ping 数据中心 10.1.3.22，直接 `Destination host unreachable`，4 个包全丢：

![宿舍区 PC3 ping 数据中心服务器，返回 Destination host unreachable 100% 丢包，背景是整网拓扑](/images/netlab-firewall-ping-test.png)

接着用扩展 ACL 放开"教学楼只能访问服务器的 www 和 ftp"。扩展 ACL 能匹配到目的端口，挂在 Fa1/0 出口：

```text
access-list 101 permit tcp 192.168.0.0 0.0.255.255 any eq www
access-list 101 permit tcp 192.168.0.0 0.0.255.255 any eq ftp
! 默认 deny any，到此教学楼只放行了 http/ftp
interface f1/0
 ip access-group 101 out
```

验证效果非常直观：教学楼 PC2 用浏览器开 `http://10.1.3.22` 正常出页面；而同一份服务器，宿舍区 PC4 开就是 `Request Timeout`。但 PC2 ping 服务器依然不通，因为扩展 ACL 只放行了 TCP 的 www/ftp，ICMP 没在放行列表里：

![左右两个 Web Browser 对比：左侧教学楼 PC2 正常打开 10.1.3.22 网页，右侧宿舍区 PC4 访问同一地址 Request Timeout](/images/netlab-acl-web-test.png)

## 七、踩坑实录

**坑 1：网关和直连接口 IP 对不上，怎么 ping 都不通**

最开始 PC 配好了 IP，区域内就是不通。排查半天才发现：和主机直连的三层交换机/路由器子接口 IP 写成了别的网段，和主机网关对不上。二层再通，三层网关错了照样黑盒。**先把"接口 IP = 主机网关"这一条对齐，再谈路由协议。**

**坑 2：ACL 只写 deny 不写 permit any，全网都断了**

标准 ACL 默认末尾是隐式 `deny any`。第一版只写了 `deny 172.16.0.0/16`，结果教学区也跟着访问不了数据中心，因为没写 `permit any`，剩下的流量全被隐式规则吞了。打完 deny 一定补 `permit any`。

**坑 3：忘了 overload，公网 IP 占满就上不了网**

一开始 NAT 没加 overload，8 个公网 IP 一对一分配，内网主机一多，新会话拿不到公网地址直接超时。加上 `overload` 做 PAT 端口复用后，同一公网 IP 能并发撑住几十条私网会话，地址资源立刻富裕了。

## 八、小结与思考

这次实验把一张学院网的全套配置串了下来：

1. **分段建设**：教学区单臂路由（子接口 dot1Q 封装）、宿舍区三层交换机 SVI，先区域互通；
2. **动态路由**：OSPF area 0 统一宣告，邻居 FULL 后全网可达；
3. **出口控制**：动态 NAT + overload 做宿舍区的公网地址复用；
4. **访问策略**：标准 ACL 拦宿舍区→数据中心，扩展 ACL 只放教学区→服务器的 www/ftp。

网络实验最爽的一点就是**策略对错立刻能被 ping 和浏览器证伪**，通了就是通了，不通就是不通，没有模糊地带。真要再往深做，我会把这套手工命令重写成脚本模板，再加上 OSPF 多 area 设计，把单骨干网扩成层次化架构。

---

*本文整理自《网络工程组网技术》组网综合实验，命令与拓扑均为 Cisco Packet Tracer 实测配置。*

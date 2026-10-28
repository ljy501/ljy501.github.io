---
title: "把一个游戏网页平台塞进 K8s：从 Minikube 搭集群到滚动更新与回滚"
date: 2026-10-28
categories: ["项目日志"]
tags: ["Kubernetes", "Docker", "Minikube", "DevOps", "Nginx"]
---

> I deployed a Nintendo-style game web platform onto a single-node Kubernetes cluster with Minikube: containerized the site into a Docker image, pushed it to a private registry, described the whole workload with a Deployment manifest, then practiced rolling updates, rollbacks, resource tuning and crash troubleshooting end to end on a laptop VM.

## 一、前言：单机虚拟机也能玩容器编排

一说 Kubernetes，很多人第一反应是"几十台服务器的集群"。但对学习者来说，根本不需要真集群，一台本地虚拟机上的 **Minikube 单节点集群**，就足够把 Deployment、Service、滚动更新、回滚这些核心概念完整跑一遍。

我这次拿一个任天堂风格的游戏网页平台当"被部署对象"，目的不是做网站本身，而是走通**一条容器化部署的标准流水线**：写 Dockerfile → 构建镜像 → 推私有仓库 → 写 Deployment YAML → `kubectl apply` → 暴露 Service → 滚动更新/回滚 → 排障。下面按实际操作顺序复盘。

## 二、环境与架构

整个实验跑在一台 Ubuntu 虚拟机里，组件很轻：

| 组件 | 作用 | 说明 |
|---|---|---|
| Docker | 容器运行时 | 构建镜像、跑容器，同时作为 Minikube 驱动 |
| Nginx | 静态 Web 服务 | 承载游戏网页平台 |
| Minikube | 单节点 K8s | 本地拉起一个可用的 Kubernetes 集群 |
| kubectl | 客户端 | 下发一切指令 |
| 阿里云镜像仓库 | 私有 registry | 存放 game:v1.0 / v1.1 镜像 |

先把集群拉起来。**这一步在国内最容易卡在网络上**，所以直接指定国内镜像源启动：

```bash
minikube start --driver=docker \
  --image-mirror-country=cn \
  --image-repository=registry.cn-hangzhou.aliyuncs.com/google_containers
```

启动后 `kubectl get nodes` 能看到单节点 Ready，就说明控制面和 kubelet 都活着，可以开始干活了。

## 三、核心实现：从镜像到 Deployment

**Step 1：把平台打包成镜像**

在 `app/game` 目录里放好平台源码，写一个 Dockerfile 用 Nginx 把静态站点跑起来：

```dockerfile
FROM nginx:alpine
COPY . /usr/share/nginx/html
EXPOSE 6065
```

构建、打标签、推到阿里云私有仓库：

```bash
docker build -t game:v1.0 .
docker tag game:v1.0 crpi-xxx.personal.cr.aliyuncs.com/lb2832/game:v1.0
docker push crpi-xxx.personal.cr.aliyuncs.com/lb2832/game:v1.0
```

私有仓库要拉镜像得先有凭证，用 `kubectl create secret docker-registry` 把仓库账号配成 imagePullSecret：**注意这一步必须在 Minikube 处于运行状态时做**，否则会直接报错。

**Step 2：用 Deployment 描述整个工作负载**

这是整个部署的核心。副本数、镜像、端口、资源配额、健康检查，全部写在一份 YAML 里：

![game-deployment.yaml 配置](/images/k8s-nintendo-deployment-yaml.jpeg)

*3 副本、容器端口 6065、requests/limits 资源配额、liveness/readiness 探针一应俱全*

对应的关键配置长这样：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: game-app
  labels: { app: game }
spec:
  replicas: 3
  selector:
    matchLabels: { app: game }
  template:
    metadata:
      labels: { app: game }
    spec:
      containers:
      - name: game-container
        image: crpi-xxx.personal.cr.aliyuncs.com/lb2832/game:v1.0
        imagePullPolicy: Always
        ports:
        - containerPort: 6065
        resources:
          requests: { memory: "256Mi", cpu: "100m" }
          limits:   { memory: "512Mi", cpu: "200m" }
        livenessProbe:
          httpGet: { path: /, port: 6065 }
          initialDelaySeconds: 30
          periodSeconds: 10
```

**Step 3：apply + 暴露 Service**

```bash
kubectl apply -f game-deployment.yaml
```

Deployment 起来后再建一个 Service，给 Pod 一个稳定的访问入口（测试环境用 NodePort 直接通过节点端口暴露）：

```yaml
apiVersion: v1
kind: Service
metadata: { name: game-service }
spec:
  selector: { app: game }
  type: NodePort
  ports:
  - port: 80
    targetPort: 6065
```

部署完去浏览器访问，游戏平台正常打开，就说明这条链路通了。

## 四、应用管理与滚动更新

起来只是开始，日常运维全靠这几条命令摸状态：

```bash
kubectl get deployment game-app     # 看副本就绪情况
kubectl get pods                     # 看每个 Pod 状态
kubectl describe deployment game-app # 看详细配置、事件
kubectl logs <pod-name>              # 看单个 Pod 日志
```

`describe` 出来的结果很说明问题：3 个副本全部 desired/updated/available，滚动更新策略、资源上下限、健康检查都清清楚楚：

![kubectl describe deployment 输出](/images/k8s-nintendo-describe.png)

*3 desired / 3 updated / 3 available，RollingUpdate 策略与 requests/limits 资源配额一目了然*

**滚动更新**：把仓库里的镜像升到 v1.1，`kubectl set image` 一改，Deployment 就自动滚动换 Pod，业务不中断：

```bash
kubectl set image deployment/game-app \
  game-container=crpi-xxx.personal.cr.aliyuncs.com/lb2832/game:v1.1
```

**回滚**：万一新版本有问题，一行命令退回上一个稳定版本：

```bash
kubectl rollout undo deployment/game-app
```

![kubectl rollout undo 回滚](/images/k8s-nintendo-rollback.png)

*rollout undo 后 describe 确认镜像已回到 v1.0*

## 五、踩坑实录

**坑 1：Minikube 起不来 / 镜像拉取失败**

现象：`minikube start` 卡住或报拉取镜像超时。原因是默认从 Google 的镜像仓库拉组件，网络不通。解决就是开头那条命令，启动时用 `--image-mirror-country=cn` + `--image-repository` 把组件镜像指向阿里云。

**坑 2：Docker 构建镜像也拉不动基础镜像**

即使 Minikube 解决了，`docker build` 拉 `nginx` 基础镜像照样慢。得给 Docker 单独配下载源，改 `/etc/docker/daemon.json` 加国内镜像加速，然后：

```bash
sudo systemctl daemon-reload
sudo systemctl restart docker
```

**坑 3：Ubuntu 自带 apt 源不稳定**

装 Minikube、kubectl 这类工具时，默认下载源经常断连。换到华为云这类稳定镜像源即可：

![更换 Ubuntu apt 下载源](/images/k8s-nintendo-apt-mirror.png)

*在"软件和更新"里把下载自换成华为云镜像，安装工具不再频频失败*

**坑 4：故意制造的故障，镜像写错版本**

把镜像改成一个不存在的版本号，Pod 立刻进入 `CrashLoopBackOff`。排障套路很固定：`kubectl describe pod` 看 Events，一眼能看到 `ImagePullBackOff` / `ErrImagePull`，确认是镜像问题后，`kubectl rollout undo` 回 v1.0 或直接把镜像改回正确版本即恢复。

## 六、资源调参与小结

除了换镜像，还练了资源配额调整：最初把内存 limit 设成 512Mi，遇到资源吃紧就改成 1Gi，改完等一会儿再看 Pod 状态才会生效：**改配置后不要立刻下结论，给控制器一点 reconcile 的时间**。

| 操作 | 命令/动作 | 结果 |
|---|---|---|
| 部署 | `kubectl apply -f game-deployment.yaml` | 3 副本全部 Ready |
| 滚动更新 | `kubectl set image ...v1.1` | 无停机换版 |
| 回滚 | `kubectl rollout undo` | 镜像回到 v1.0 |
| 故障恢复 | describe 看 Events + 回滚 | Crash 的 Pod 恢复正常 |

这次实践下来，对 K8s 的理解从"一堆 YAML"变成了"一条可操作的运维流水线"：Deployment 声明期望状态、控制器负责收敛、Service 负责稳定入口、回滚是兜底。单机 Minikube 虽然不能代表生产集群，但把这些核心动作亲手做过一遍，再去看三节点、Ingress、HPA 那些东西就顺理成章了。

---

*本文整理自一次 Kubernetes 部署与基础运维实训，命令与截图均为 Ubuntu 虚拟机 + Minikube 实测。*

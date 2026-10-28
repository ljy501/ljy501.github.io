---
title: "从容器化 GitLab 到 CRUD 自动生成：企业级 JavaEE 应用二开全流程"
date: 2026-10-28
categories: ["项目日志"]
tags: ["GitLab", "Docker", "Spring Boot", "Vue3", "MyBatis-Flex"]
---

> This lab walked the full chain of building an enterprise JavaEE web system: containerize a self-hosted GitLab on CentOS 7.6, assemble a JDK21 + MySQL8 + Redis dev environment, fork the open-source Sz-Admin project (Spring Boot 3 + MyBatis-Flex + Vue3) onto the team's repos, then use its built-in code generator to produce full CRUD modules for two business tables in minutes.

## 一、前言：企业级开发不是写接口，是先搭"流水线"

学校里做课设，习惯了单枪匹马写个 Spring Boot 就交差。但真实的企业级项目上来先面对一堆"基建"问题：**代码放哪、团队怎么协作、本地环境怎么统一、新表的增删改查能不能不手写**。

这次实训的目标就是把这条流水线从头到尾走一遍：先在虚拟机里用 Docker 把 GitLab 跑起来当团队仓库，再配齐 JDK21 / MySQL8 / Redis 的开发环境，然后基于开源后台管理框架 Sz-Admin 做二次开发，收尾用它自带的代码生成器，把两张业务表直接生成完整的前后端 CRUD。

## 二、技术栈全景：一张表看清整条链

| 组件 | 版本/选型 | 用途 |
|---|---|---|
| CentOS 7.6 虚拟机 | VMware | 跑 GitLab 的服务器 |
| Docker CE | 最新稳定版 | 容器化部署 GitLab |
| GitLab | 官方 Docker 镜像 | 团队代码托管 |
| JDK | Zulu 21 (21.0.7 LTS) | Spring Boot 3 运行底座 |
| MySQL | 8.0 | 业务数据库 |
| Redis | Windows 版 3.0.504 | 缓存/会话 |
| IntelliJ IDEA | 2024.3.1.1 | 前后端统一开发 |
| Sz-Admin | 开源 RBAC 后台（Spring Boot 3 + MyBatis-Flex + Vue3） | 二开底座 |
| pnpm | - | 前端依赖安装 |

## 三、Step 1：Docker 一把梭部署 GitLab

团队协作的第一块砖是代码托管。直接装系统级 GitLab 太脏，我选择在 CentOS 虚拟机里用 Docker 跑：

```bash
# 安装 docker 引擎
sudo yum install -y docker-ce docker-ce-cli containerd.io \
     docker-buildx-plugin docker-compose-plugin

# 开机自启 + 立即启动
sudo systemctl enable docker
sudo systemctl start docker
```

GitLab 的配置、日志、数据必须落在宿主机卷上，不然容器一删全没。先建好数据目录、挂上 `$GITLAB_HOME` 环境变量：

```bash
sudo mkdir /srv/gitlab
# 写入 /etc/profile
echo 'export GITLAB_HOME=/srv/gitlab' | sudo tee -a /etc/profile
source /etc/profile
```

然后装 Docker Compose、拉 GitLab 镜像、一条命令起服务：

```bash
sudo docker-compose up -d
```

这一步要做好心理准备：**首次启动可能要 30 分钟甚至更久**，取决于镜像拉取速度。看到 GitLab 容器 healthy 之后，浏览器访问 `http://192.168.x.x:8090`，初始账号 root，首次登录自己设密码，团队仓库就上线了。

## 四、Step 2：本地开发环境四件套

GitLab 管代码，本地环境管运行。JDK21 是 Spring Boot 3 的硬门槛（Boot 3 要求 17+），先验证版本：

![java -version 验证 JDK 21.0.7 LTS](/images/webapp-jdk-verify.png)

*Zulu 21.0.7 LTS，HotSpot 64-Bit Server VM，Spring Boot 3 的门槛过了*

Redis 直接用 Windows 版 3.0.504 的 msi 安装，装完在服务里确认它在跑，再用 `redis-cli` 验证连通，然后改配置文件加密码：

```powershell
# 修改 C:\Program Files\Redis\redis.windows-service.conf
# requirepass 123456，改完重启 Redis 服务生效
```

MySQL 这边重点是**把账号权限模型建对**，建独立业务账号、独立业务库，而不是拿 root 到处连：

```sql
-- 用 root 登录后操作
use mysql;

-- 建本地业务用户
create user 'ljy'@'localhost' identified by '*******';
create database db4ljy;
grant all privileges on db4ljy.* to 'ljy'@'localhost';
flush privileges;

-- 需要远程访问再开 % 主机位
create user 'ljy'@'%' identified by '*******';
grant all privileges on db4ljy.* to 'ljy'@'%';
flush privileges;
```

这一套"一用户一库一授权"的范式，后来在 Sz-Admin 连库时直接复用，没有再出过权限问题。

## 五、Step 3：Fork 开源底座，团队并行开发

环境齐了，开始拉项目。Sz-Admin 在 Gitee 上有三个模块：后端父工程 `sz-boot-parent`、前端 `sz-admin`、部署文档 `sz-deploy`。组内选一个人把三个仓库全部 Fork 到自己的私有空间，再把组员一个个加成"开发者"：**每个人都在同一个 Fork 仓库上开发，权限互不越界**。

后端在 IDEA 里 `Project from Version Control` 克隆 `sz-boot-parent`，按官方文档从"修改配置文件"开始调，重点是把数据库连接、Redis 地址、端口这几项和本地环境对齐。跑起来后后端接口文档挂在 `http://127.0.0.1:9991/api/admin/doc.html`，浏览器能打开这份 Swagger 文档，就说明后端服务、数据库连接、接口扫描整条链路是通的。

前端克隆 `sz-admin` 后第一件事往往是**发现自己没装 Node.js**，装完再用 pnpm 装依赖：

![pnpm install 一键装完 658 个前端依赖](/images/webapp-frontend-pnpm.png)

*Lockfile 命中，658 个包一次装齐，element-plus、typescript 全部 done*

前端跑起来后访问 `http://localhost:9848/login`，用默认账号登录，整个 RBAC 后台就展现在眼前了。到这一步，"团队有仓库 + 前后端都能跑"的基建全部就绪。

## 六、Step 4：代码生成器，两张表，十分钟出 CRUD

真正体现二开效率的是 Sz-Admin 内置的代码生成器。流程高度标准化：

**第一步**，先在数据库里建好业务表（比如 `teacher_statistics` 和 `student4liujy`）。

**第二步**，改后端配置 `application.yml`，告诉生成器前端项目在磁盘的哪个目录、代码作者是谁：

```yaml
generator:
  path:
    web: C://Users//...//IdeaProjects//sz-admin   # 前端项目本地路径
    api:  E://code//Gitlab//...//sz-admin          # 后端默认自动检测
  module-name: sz-service
  service-name: sz-service-admin
global:
  author: liujy
  packages: com.sz.admin
```

**第三步**，后台进"工具箱 → 代码生成"，导入目标表，一步步走三步向导：

![代码生成器第一步：基本信息，设置表描述与实体类名](/images/webapp-codegen-dialog.png)

*表名 teacher → 实体类 Teacher，表描述"教师表"，作者 sz-admin*

每个字段还能精细配置：是不是主键、要不要逻辑删除、列表/查询/导入/导出哪些实体包含它、查询方式是精确还是模糊、前端展示用文本框还是下拉框、挂哪个字典：**生成出来的代码质量上限，全在这一步定**。

这里的取舍值得多说两句。比如"手机号"字段如果配成模糊查询 + 文本框，生成出来的列表页就能按手机号片段搜；如果"年级"这种枚举值配成字典 + 下拉框，前端自动出筛选下拉，后端自动做字典翻译。字段属性配得越细，生成的 CRUD 就越接近真实业务需要，后期返工越少。

点"生成代码"，系统会自动做两件保护：目标路径已有同名文件**不覆盖**；菜单重复则**不重复注册**。生成完别着急刷新，新菜单要在角色管理里给角色分配权限，而且新代码还没被 JVM 加载，**必须重启后端服务**才生效。

两张表各跑一遍向导，最终生成的业务页面长这样：

![自动生成的教师表 CRUD 页面：搜索、新增、批量删除、导入、导出全套](/images/webapp-crud-page.png)

*姓名/年龄/年级组合查询，新增、批量删除、导入、导出按钮齐全，一张表从建库到页面，手写代码量接近于零*

## 七、踩坑实录：这三个坑我都替你踩过了

**坑 1：`docker-compose up -d` 卡了半小时没动静**

第一次跑以为环境坏了。实际就是镜像在慢网络里慢慢拉。判断方法：`docker ps` 看容器状态，`docker logs gitlab` 看初始化日志。真出问题就 `docker restart gitlab` 重启容器，配置全在 `/srv/gitlab` 卷里，不丢。

**坑 2：前端 pnpm install 直接报错**

报错信息指向 Node 缺失。Sz-Admin 是 Vue3 + Vite 工程，本地必须先有 Node.js 环境。装完 Node 再跑 pnpm，658 个包一次过。

**坑 3：代码生成完，菜单死活不出现**

两个叠加原因：一是**没给角色分配新菜单权限**，admin 之外的角色根本看不见；二是 **JVM 没重启**，生成的 Java 类还没被类加载器加载。两个动作都做：角色管理勾权限 → 重启后端 → 刷新页面。

## 八、成果清单：这条流水线最终交付了什么

| 环节 | 成果 | 验证方式 |
|---|---|---|
| 代码托管 | Docker 化 GitLab 团队仓库 | 浏览器访问 8090 端口 |
| 后端 | sz-boot-parent 运行中 | `doc.html` 接口文档可访问 |
| 前端 | sz-admin 运行中 | `localhost:9848` 登录页 |
| 业务模块 | teacher_statistics、student4liujy 两张表的完整 CRUD | 生成页面可增删改查、导入导出 |

## 九、总结与思考

这次实训最大的收获不是写了多少代码，而是**把企业级开发的"地基"亲手搭了一遍**：容器化解决环境一致性，Fork + 成员权限解决协作边界，代码生成器解决重复劳动。

回头看，传统的"建表 → 写实体 → 写 Mapper → 写 Service → 写 Controller → 写前端页面"这一整套手工活，在成熟的开源框架里已经被压缩成"建表 + 配字段 + 重启"三步。二开的真正价值，应该花在业务逻辑上，而不是 CRUD 样板上。

---

*本文整理自《企业级 Web 应用系统构建与开发》实训，环境版本与工具链均为实操记录。*

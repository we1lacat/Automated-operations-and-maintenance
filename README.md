# Automated Operations and Maintenance

> 运维方向学习笔记仓库，涵盖从单机到编排的完整技术链路：**Linux 基础 → Shell 编程 → Docker 容器化 → Kubernetes 编排 → 可观测性**。  
> 所有命令与配置均在自有主机与多节点集群（RHEL 10 / K3s / Kubeadm）实操验证。

## 目录

- [仓库内容](#仓库内容)
- [建议阅读顺序](#建议阅读顺序)
- [后续计划](#后续计划)
- [笔记版式约定](#笔记版式约定)
- [版本基线](#版本基线)
- [写在最后](#写在最后)

---

## 仓库内容

```text
Linux/           Linux 基础、Shell 编程、排障手册与 Ansible 自动化
container/       Docker 容器化与 Kubernetes 编排
common server/   常用基础服务（NFS 等）
Log system/      监控指标与可观测体系
```

| 文件 | 行数 | 核心覆盖内容 |
| :--- | :--- | :--- |
| [`Linux/Linux-note.md`](Linux/Linux-note.md) | 3326 | **权限与安全**（chmod/chown、umask、ACL、SELinux）、**文本三剑客**（grep/sed/awk、正则）、**磁盘与存储**（fdisk/GPT、xfs/ext4、fstab、LVM 扩容、autofs、NFS）、**进程与服务**（systemd、journalctl、tuned、前后台与会话持久化）、**软件与网络**（dnf/源码编译/Flatpak、nmcli、firewalld、SSH）、**Shell 基础** |
| [`Linux/Linux-note-fixlog.md`](Linux/Linux-note-fixlog.md) | — | 笔记结构性问题处理复盘：问题清单、根因分析、踩坑沉淀与维护检查清单 |
| [`Linux/troubleshooting.md`](Linux/troubleshooting.md) | 397 | **Linux 排障手册（按症状反查）**：权限、文本处理、磁盘与 LVM（含 fstab 污染 SELinux 标签导致只读事故复盘）、进程服务、防火墙与网络、长任务会话 |
| [`Linux/ansible.md`](Linux/ansible.md) | 1273 | **Ansible 自动化（RHCE EX294 / RHEL 10）**：架构与配置优先级、Inventory 与提权、Playbook 与幂等验证、22 级变量优先级与 Vault、控制流、十大常用模块、Roles 与 RHEL System Roles、并发滚动与排障 |
| [`Linux/shell-note-zh.md`](Linux/shell-note-zh.md) | 964 | **Shell 编程（中文版）**：Bash 特性、父/子 Shell 执行环境、变量与字符串运算、条件/循环与函数开发（另有英文原稿 [`shell.md`](Linux/shell.md)，842 行） |
| [`container/docker-note.md`](container/docker-note.md) | 2462 | 镜像分层原理、常用命令、数据卷、Dockerfile 构建与缓存优化、Docker 网络（docker0 / veth pair / 自定义网络）、Compose、Swarm 集群与 Secret |
| [`container/k8s-note.md`](container/k8s-note.md) | 3550 | 架构与 kubeadm 部署（Calico/Dashboard/k9s）、核心资源（Pod/Deployment/Service/Ingress/存储/调度）、kubectl 排障工具箱、进阶专题（HPA、StatefulSet、Job/CronJob、RBAC 安全与 CI/CD 凭证、Helm、证书续期与 etcd 备份） |
| [`common server/NFS.md`](common%20server/NFS.md) | 1018 | RPC/rpcbind 端口注册原理、NFSv2–v4.2 演进、部署与 SELinux/防火墙、`/etc/exports` 全参数、fstab/autofs 自动挂载、`root_squash` 权限模型、14 条排障速查、与 K8s PV/StorageClass 联动 |
| [`Log system/Prometheus.md`](Log%20system/Prometheus.md) | 1788 | 四类可观测数据选型、数据模型与高基数代价、Pull 机制与 Exporter、TSDB 存储与内存估算、PromQL 算子与陷阱、K8s 部署（`kube-prometheus-stack` 2C2G 降配、ServiceMonitor 坑点）、Grafana 迁移、双机隧道监控实战、Alertmanager 路由抑制与排障 |
| `LICENSE` | — | MIT License，Copyright (c) 2026 we1l |

---

## 建议阅读顺序

```text
① Linux 基础 ──▶ ② Shell 编程 ──▶ ③ Docker ──▶ ④ Kubernetes ──▶ ⑤ 可观测
```

1. **Linux 基础**：掌握单机目录、权限、进程与服务，是后续所有排障的地基（如 Pod 网络不通往往源于宿主机 iptables 桥接或变量未加引号）。
2. **Shell 编程**：自动化脚本入口（`shell-note-zh.md` 与 `shell.md` 内容同源，读一份即可）。
3. **Docker**：理解镜像分层、卷持久化与容器网络隔离。
4. **Kubernetes**：掌握声明式 API、控制器循环与集群排障。
5. **可观测性**：建立指标、日志与告警闭环。

---

## 后续计划

| 方向 | 待补内容 |
| :--- | :--- |
| **常用服务** | 堡垒机、DNS、邮件服务器（NFS 已整理） |
| **Web 服务** | Nginx、Tomcat、LVS + Keepalived + HAProxy |
| **数据库** | MySQL、Redis 部署、备份与故障处理 |
| **自动化** | Jenkins / CI-CD 流水线（Ansible 已整理） |
| **可观测** | **EFK / Loki 日志栈**、链路追踪 Jaeger（Prometheus + Grafana 已整理） |

> 整理进度视实操节奏不定期更新，仓库目录即当前最新进度。

---

## 笔记版式约定

- **表格优先**：概念对比、命令速查、参数说明统一采用表格，便于横向检索。
- **示例可抄**：命令与 YAML 均附注释，明确标注需修改的字段与常见坑点。
- **引用块纠偏（`>`）**：集中记录版本差异、生产事故复盘与高频误用。
- **每章小结**：提炼章节主干，支持快速复习与排障反查（直接搜索 `ImagePullBackOff`、`CrashLoopBackOff`、`OOMKilled`、`403 forbidden` 等症状关键词）。

---

## 版本基线

| 组件 | 版本与环境说明 |
| :--- | :--- |
| **Kubernetes** | 引导部署示例（§3.1–3.4）基于 `kubeadm v1.20.9`；集群实战、安全与组件（§3.5 / §5–6）均在 **v1.33.x** 实操验证，1.27+ 破坏性变更已在正文单独标注 |
| **CNI / Ingress** | `Calico v3.31.6`；`ingress-nginx v0.46.0`（2026-03 已退役，新集群建议迁移至 Gateway API，正文 §6.1 已注） |
| **Dashboard / TUI** | Helm v3.x 部署（Dashboard 项目于 2026-01 归档，继任者为 Headlamp，正文已注）；终端管理工具 `k9s v0.51.0` |
| **Docker** | `Docker Engine 27.3.1`、`Compose v2.29.1` |
| **镜像源** | 示例默认配置国内镜像源（阿里云等），海外环境按需替换即可 |

> **注意**：本仓库为个人实战笔记，非官方文档替代品；涉及数据清理、证书更新或 etcd 恢复等高危操作，请务必先在测试环境验证。

---

## 写在最后

> AI 加速了资料检索与结构整理，但运维实战没有捷径——仓库里的每一条命令、每一份配置都在自己的机器上跑通并排过坑。  
> 技术更迭很快，保持动手与热爱，愿这些记录也能帮到正在路上的人。

[MIT License](LICENSE) © 2026 we1l

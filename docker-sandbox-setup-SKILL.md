---
name: docker-sandbox-setup
description: "Configure and implement Docker sandbox for Hermes Agent — enable Docker backend, set resource limits, apply security policies, verify isolation. For WSL2 + Docker Desktop environments."
tags: [docker, sandbox, setup, security, hermes, wsl2]
triggers:
  - "setup docker sandbox"
  - "configure docker backend"
  - "docker沙箱配置"
  - "启用docker后端"
  - "docker安全策略"
related_skills: [docker-sandbox-testing, hermes-agent]
---

# Docker Sandbox Setup

为 Hermes Agent 配置 Docker 沙箱执行环境，实现容器隔离、资源限制、安全策略。

## 前置条件

- Docker Desktop 已安装（Windows 侧）
- WSL2 集成已启用
- Docker 命令可在 WSL 中执行

## 配置步骤

### Step 1: 验证 Docker 可用性

```bash
docker --version
docker ps
```

如果 `permission denied`，启用 WSL 集成：
- Docker Desktop → Settings → Resources → WSL Integration → 勾选你的发行机 → Apply & Restart

### Step 2: 修改 Hermes 配置

编辑 `~/.hermes/config.yaml`：

```yaml
terminal:
  backend: docker
  docker_image: nikolaik/python-nodejs:python3.11-nodejs20
  container_cpu: 1
  container_memory: 4096
  container_disk: 20480
  docker_volumes: []
  docker_mount_cwd_to_workspace: false
  docker_run_as_host_user: true
  persistent_shell: false
  lifetime_seconds: 600
```

### Step 3: 创建安全策略

创建 `~/.hermes/docker-security-policy.json`：

```json
{
  "security_opt": ["no-new-privileges"],
  "cap_drop": ["ALL"],
  "cap_add": [],
  "read_only": true,
  "tmpfs": {
    "/tmp": "rw,noexec,nosuid,size=100m"
  },
  "network_mode": "none",
  "pids_limit": 100,
  "ulimits": {
    "nofile": { "Name": "nofile", "Hard": 1024, "Soft": 1024 }
  }
}
```

### Step 4: 重启会话

```bash
# 在 CLI 中
/reset

# 或重启 gateway
hermes gateway restart
```

## 验证清单

| 检查项 | 命令 | 预期结果 |
|--------|------|----------|
| Docker后端 | `grep "backend: docker" ~/.hermes/config.yaml` | 找到匹配 |
| 内存限制 | `cat /sys/fs/cgroup/memory.max` | 显示字节数 |
| CPU限制 | `cat /sys/fs/cgroup/cpu.max` | `100000 100000` |
| 网络隔离 | `ping -c 1 8.8.8.8` | Network unreachable |
| 文件只读 | `touch /test` | Read-only file system |
| 权限限制 | `cat /proc/self/status \| grep Cap` | 全为0 |

## 常见问题

### Docker socket 权限

```bash
# 临时修复
sudo chmod 666 /var/run/docker.sock

# 永久修复
sudo usermod -aG docker $USER && newgrp docker
```

### nproc 显示主机 CPU 数

这是正常的，`nproc` 读取 `/proc/cpuinfo` 而非 cgroup。验证 CPU 限制用：
```bash
cat /sys/fs/cgroup/cpu.max
```

### Docker Desktop WSL2 找不到 Docker

启用 WSL 集成：Settings → Resources → WSL Integration → toggle ON

### 镜像拉取失败

`nikolaik/python-nodejs:python3.11-nodejs20` 在 Docker 29.x 可能失败。测试用 `alpine:latest`。

## 配置说明

| 配置项 | 值 | 作用 |
|--------|-----|------|
| `backend` | docker | 启用 Docker 后端 |
| `container_memory` | 4096 | 内存限制 4GB |
| `container_disk` | 20480 | 磁盘限制 20GB |
| `container_cpu` | 1 | CPU 核心数 |
| `docker_run_as_host_user` | true | 以宿主用户身份运行 |
| `persistent_shell` | false | 每次创建新容器 |
| `lifetime_seconds` | 600 | 容器最大存活 10 分钟 |
| `read_only` | true | 文件系统只读 |
| `network_mode` | none | 网络隔离 |
| `cap_drop` | ALL | 移除所有 capabilities |
| `pids_limit` | 100 | 进程数限制 |

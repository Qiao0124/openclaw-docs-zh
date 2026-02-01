---
summary: "使用 Ansible、Tailscale 与防火墙隔离的自动化加固安装"
read_when:
  - 你想自动化部署并进行安全加固
  - 你需要通过 VPN 访问的防火墙隔离部署
  - 你在远程 Debian/Ubuntu 服务器上部署
title: "Ansible 安装（Ansible Installation）"
---

# Ansible 安装（Ansible Installation）

推荐在生产服务器上部署 OpenClaw 的方式是使用 **[openclaw-ansible](https://github.com/openclaw/openclaw-ansible)** —— 一个以安全为核心的自动化安装器。

## 快速开始（Quick Start）

一行命令安装：

```bash
curl -fsSL https://raw.githubusercontent.com/openclaw/openclaw-ansible/main/install.sh | bash
```

> **📦 完整指南：[github.com/openclaw/openclaw-ansible](https://github.com/openclaw/openclaw-ansible)**
>
> openclaw-ansible 仓库是 Ansible 部署的权威来源。本页仅作快速概览。

## 你将获得（What You Get）

- 🔒 **防火墙优先安全**：UFW + Docker 隔离（只允许 SSH + Tailscale）
- 🔐 **Tailscale VPN**：不暴露公网即可安全远程访问
- 🐳 **Docker**：隔离的沙箱容器，默认仅绑定 localhost
- 🛡️ **纵深防御**：4 层安全架构
- 🚀 **一键部署**：几分钟完成完整部署
- 🔧 **Systemd 集成**：开机自启并加固

## 要求（Requirements）

- **OS**：Debian 11+ 或 Ubuntu 20.04+
- **权限**：Root 或 sudo
- **网络**：安装依赖所需的互联网连接
- **Ansible**：2.14+（快速安装脚本会自动安装）

## 安装内容（What Gets Installed）

Ansible playbook 会安装并配置：

1. **Tailscale**（安全远程访问的 mesh VPN）
2. **UFW 防火墙**（仅开放 SSH + Tailscale 端口）
3. **Docker CE + Compose V2**（用于代理沙箱）
4. **Node.js 22.x + pnpm**（运行时依赖）
5. **OpenClaw**（宿主机运行，非容器化）
6. **Systemd 服务**（自动启动并加固）

注意：网关 **直接运行在宿主机**（不在 Docker 中），但代理沙箱使用 Docker 进行隔离。详见 [Sandboxing](/gateway/sandboxing)。

## 安装后设置（Post-Install Setup）

安装完成后，切换到 openclaw 用户：

```bash
sudo -i -u openclaw
```

后续脚本会引导你完成：

1. **引导向导**：配置 OpenClaw
2. **提供方登录**：连接 WhatsApp/Telegram/Discord/Signal
3. **网关测试**：验证安装
4. **Tailscale 设置**：连接 VPN mesh

### 常用命令（Quick commands）

```bash
# 查看服务状态
sudo systemctl status openclaw

# 查看实时日志
sudo journalctl -u openclaw -f

# 重启网关
sudo systemctl restart openclaw

# 提供方登录（以 openclaw 用户运行）
sudo -i -u openclaw
openclaw channels login
```

## 安全架构（Security Architecture）

### 4 层防御（4-Layer Defense）

1. **防火墙（UFW）**：仅暴露 SSH（22）+ Tailscale（41641/udp）
2. **VPN（Tailscale）**：网关仅在 VPN mesh 内可访问
3. **Docker 隔离**：DOCKER-USER iptables 链阻止外部端口暴露
4. **Systemd 加固**：NoNewPrivileges、PrivateTmp、非特权用户

### 验证（Verification）

测试对外攻击面：

```bash
nmap -p- YOUR_SERVER_IP
```

应只显示 **端口 22**（SSH）开放。其他服务（网关、Docker）都被锁定。

### Docker 可用性（Docker Availability）

Docker 仅用于 **代理沙箱**（隔离工具执行），不是用于运行网关本体。网关默认只绑定 localhost，并通过 Tailscale VPN 访问。

沙箱配置详见：[Multi-Agent Sandbox & Tools](/multi-agent-sandbox-tools)。

## 手动安装（Manual Installation）

如果你更喜欢手动控制而非自动化：

```bash
# 1. 安装前置依赖
sudo apt update && sudo apt install -y ansible git

# 2. 克隆仓库
git clone https://github.com/openclaw/openclaw-ansible.git
cd openclaw-ansible

# 3. 安装 Ansible collections
ansible-galaxy collection install -r requirements.yml

# 4. 运行 playbook
./run-playbook.sh

# 或直接运行（然后手动执行 /tmp/openclaw-setup.sh）
# ansible-playbook playbook.yml --ask-become-pass
```

## 更新 OpenClaw（Updating OpenClaw）

Ansible 安装器会把 OpenClaw 配置成手动更新。标准更新流程见 [Updating](/install/updating)。

如需重新运行 Ansible playbook（例如调整配置）：

```bash
cd openclaw-ansible
./run-playbook.sh
```

说明：该流程具备幂等性，可安全多次运行。

## 故障排查（Troubleshooting）

### 防火墙阻断连接

如果你被锁在外面：

- 确保先通过 Tailscale VPN 访问
- SSH（22）始终允许
- 网关 **默认只允许** 通过 Tailscale 访问

### 服务无法启动

```bash
# 查看日志
sudo journalctl -u openclaw -n 100

# 验证权限
sudo ls -la /opt/openclaw

# 测试手动启动
sudo -i -u openclaw
cd ~/openclaw
pnpm start
```

### Docker 沙箱问题

```bash
# 确认 Docker 正常运行
sudo systemctl status docker

# 检查沙箱镜像
sudo docker images | grep openclaw-sandbox

# 如果缺失则构建沙箱镜像
cd /opt/openclaw/openclaw
sudo -u openclaw ./scripts/sandbox-setup.sh
```

### 提供方登录失败

请确认以 `openclaw` 用户运行：

```bash
sudo -i -u openclaw
openclaw channels login
```

## 高级配置（Advanced Configuration）

更详细的安全架构与排障说明：

- [Security Architecture](https://github.com/openclaw/openclaw-ansible/blob/main/docs/security.md)
- [Technical Details](https://github.com/openclaw/openclaw-ansible/blob/main/docs/architecture.md)

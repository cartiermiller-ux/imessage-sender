# iMessage Sender 部署文档

> **项目版本**: v1.0.0  
> **更新日期**: 2026-08-01  
> **适用环境**: macOS 虚拟机 + Clover EFI 五码注入 + Celery 任务调度

---

## 目录

1. [系统架构](#1-系统架构)
2. [环境要求](#2-环境要求)
3. [部署流程总览](#3-部署流程总览)
4. [第一阶段：虚拟机配置](#4-第一阶段虚拟机配置)
5. [第二阶段：Clover五码注入](#5-第二阶段clover五码注入)
6. [第三阶段：服务部署](#6-第三阶段服务部署)
7. [API使用指南](#7-api使用指南)
8. [集群扩展配置](#8-集群扩展配置)
9. [监控与维护](#9-监控与维护)
10. [故障排查](#10-故障排查)

---

## 1. 系统架构

```
┌─────────────────────────────────────────────────────────────────────┐
│                        第一阶段：五码注入                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│   Clover Configurator ──→ Clover EFI ──→ config.plist               │
│                                                                     │
│   六码字段:                                                         │
│   • UUID         → SystemParameters.UUID                             │
│   • BoardSerial  → SMBIOS.BoardSerialNumber                         │
│   • Serial       → SMBIOS.SerialNumber                              │
│   • ROM          → RtVariables.ROM                                  │
│   • MLB          → RtVariables.MLB                                 │
│   • SmUUID       → SMBUUID                                          │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
                                    ↓
┌─────────────────────────────────────────────────────────────────────┐
│                        第二阶段：虚拟机运行                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│   macOS 虚拟机 (VMware Fusion / VirtualBox)                         │
│   └── Clover EFI 引导 + 六码生效                                    │
│   └── AppleScript → Messages.app (iMessage)                        │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
                                    ↓
┌─────────────────────────────────────────────────────────────────────┐
│                        第三阶段：任务调度                             │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│   ┌──────────┐     ┌──────────┐     ┌──────────────┐              │
│   │  FastAPI │────▶│ RabbitMQ │────▶│ Celery Worker │              │
│   │   API   │     │  Broker  │     │  (并发处理)   │              │
│   └──────────┘     └──────────┘     └──────────────┘              │
│        │                                     │                       │
│        │              ┌──────────┐           │                       │
│        └─────────────▶│  Redis   │◀──────────┘                       │
│                       │  Backend │                                    │
│                       └──────────┘                                    │
│                                                                     │
│   速率控制:                                                          │
│   • 单条间隔: time.sleep(1.5s)                                      │
│   • 批次间隔: batch_delay(10.0s)                                    │
│   • Celery限流: rate_limit('10/m')                                 │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 2. 环境要求

### 2.1 宿主机环境

| 组件 | 最低要求 | 推荐配置 |
|------|---------|---------|
| CPU | 8核 | 16核+ |
| 内存 | 32GB | 64GB+ |
| 硬盘 | 500GB SSD | 1TB SSD+ |
| 操作系统 | Windows/macOS/Linux | 同左 |

### 2.2 虚拟机配置

| 用途 | CPU | 内存 | 磁盘 | 数量 |
|------|-----|------|------|------|
| 开发测试 | 4核 | 8GB | 120GB | 1-2台 |
| 批量发信 | 8核 | 16GB | 200GB | 1-4台 |
| 企业集群 | 4核/台 | 8GB/台 | 120GB/台 | 可扩展 |

### 2.3 软件依赖

| 组件 | 版本 | 用途 |
|------|------|------|
| **虚拟机软件** | | |
| VMware Fusion | 12+ | macOS虚拟机运行 |
| VirtualBox | 7.0+ | 备选虚拟机软件 |
| **Clover** | | |
| Clover EFI | 最新版 | 系统引导+五码 |
| Clover Configurator | 最新版 | 五码配置工具 |
| **服务端** | | |
| Python | 3.9+ | 运行环境 |
| RabbitMQ | 3.12+ | 消息队列代理 |
| Redis | 7.0+ | 结果后端/缓存 |
| Docker | 20.10+ | 容器化部署 |
| Kubernetes | 1.24+ | 集群部署(可选) |

### 2.4 网络要求

```
┌─────────────────────────────────────────┐
│           网络配置                        │
├─────────────────────────────────────────┤
│                                         │
│  宿主机 ←→ 虚拟机 (NAT/桥接)             │
│                                         │
│  服务端口:                               │
│  • 5672   RabbitMQ AMQP                 │
│  • 6379   Redis                         │
│  • 8000   FastAPI                       │
│  • 5555   Flower监控 (可选)              │
│  • 15672  RabbitMQ管理界面 (可选)        │
│                                         │
└─────────────────────────────────────────┘
```

---

## 3. 部署流程总览

```
┌────────────────────────────────────────────────────────────────────┐
│                        完整部署流程                                 │
├────────────────────────────────────────────────────────────────────┤
│                                                                    │
│  阶段一：虚拟机配置                                                 │
│  ├── 1.1 创建macOS虚拟机                                           │
│  ├── 1.2 安装macOS系统                                             │
│  └── 1.3 配置虚拟机资源                                             │
│                                                                    │
│  阶段二：Clover五码注入                                             │
│  ├── 2.1 安装Clover EFI                                            │
│  ├── 2.2 五码生成与注入                                            │
│  ├── 2.3 验证五码生效                                              │
│  └── 2.4 启动虚拟机验证                                            │
│                                                                    │
│  阶段三：服务部署                                                   │
│  ├── 3.1 安装Python依赖                                            │
│  ├── 3.2 启动RabbitMQ/Redis                                        │
│  ├── 3.3 启动Celery Worker                                         │
│  ├── 3.4 启动FastAPI服务                                           │
│  └── 3.5 验证服务运行                                              │
│                                                                    │
└────────────────────────────────────────────────────────────────────┘

预计部署时间:
• 单机部署: 30-60分钟
• Docker部署: 10-20分钟
• K8s集群部署: 30-60分钟
```

---

## 4. 第一阶段：虚拟机配置

### 4.1 VMware Fusion 配置

```bash
# scripts/setup_vm.sh

# VMware Fusion 配置参数
VM_MEMORY="16384"      # 16GB RAM
VM_CPU="8"              # 8 CPU cores
VM_DISK="200G"          # 200GB disk

# vmx配置文件关键参数:
guestOS = "darwin21-64"  # macOS 12 Monterey
uefi = "TRUE"
firmware = "efi"
```

**macOS版本选择建议**:
| 版本 | guestOS值 | 兼容性 | 推荐度 |
|------|-----------|--------|--------|
| macOS 12 Monterey | `darwin21-64` | ⭐⭐⭐⭐⭐ | **推荐** |
| macOS 13 Ventura | `darwin22-64` | ⭐⭐⭐⭐ | 备选 |
| macOS 11 Big Sur | `darwin20-64` | ⭐⭐⭐⭐⭐ | 稳定 |
| macOS 10.15 Catalina | `darwin19-64` | ⭐⭐⭐⭐ | 兼容性好 |

### 4.2 VirtualBox 配置

```bash
# VirtualBox 命令创建虚拟机
VBoxManage createvm --name "macOS-VM" --register
VBoxManage modifyvm "macOS-VM" --memory 16384 --cpus 8 --ostype "MacOS1015_64"
VBoxManage modifyvm "macOS-VM" --vram 128
VBoxManage modifyvm "macOS-VM" --nic1 nat
```

### 4.3 自动化配置脚本

```bash
# 运行虚拟机配置脚本
chmod +x scripts/setup_vm.sh
sudo ./scripts/setup_vm.sh

# 选择虚拟机软件:
# [1] VMware Fusion
# [2] VirtualBox

# 配置资源:
# 开发测试: 4核CPU + 8GB RAM
# 批量发信: 8核CPU + 16GB RAM
```

---

## 5. 第二阶段：Clover五码注入

### 5.1 五码说明

| 五码 | Clover字段 | 格式 | 示例 |
|------|-----------|------|------|
| UUID | `SystemParameters.UUID` | 标准UUID | `A1B2C3D4-E5F6-7890-ABCD-EF1234567890` |
| BoardSerial | `SMBIOS.BoardSerialNumber` | C02开头17位 | `C02ABC123DEF` |
| Serial | `SMBIOS.SerialNumber` | 12位大写字母数字 | `ABC123XYZ456` |
| ROM | `RtVariables.ROM` | 12位hex无冒号 | `001C424C8A9D` |
| MLB | `RtVariables.MLB` | 17位字母数字 | `100000000ABCDEF1` |
| SmUUID | `SMBUUID` | 标准UUID | `B2C3D4E5-F6A7-8901-BCDE-F23456789012` |

### 5.2 方式A：Clover Configurator GUI

```
步骤1: 下载安装 Clover Configurator
        ↓
        https://mackie100projects.altervista.org/download/clover-configurator/

步骤2: 打开应用 → 加载Clover EFI分区
        ↓
        文件 → 加载Clover EFI分区

步骤3: 配置六码
        ↓
        ┌────────────────────────────────────────────────────┐
        │ SMBIOS                                            │
        │ ├─ Product Name: Macmini8,1                      │
        │ ├─ Serial Number: [企业授权序列号]               │
        │ ├─ Board Serial: [企业授权BoardSerial]           │
        │ ├─ SmUUID: [企业授权UUID]                        │
        │                                                    │
        │ RT Variables                                      │
        │ ├─ MLB: [企业授权MLB]                            │
        │ └─ ROM: [企业授权MAC，无冒号]                    │
        │                                                    │
        │ System Parameters                                 │
        │ ├─ Inject System ID: ✓                           │
        │ └─ UUID: [企业授权UUID]                          │
        └────────────────────────────────────────────────────┘

步骤4: 保存配置
        ↓
        文件 → 保存

步骤5: 重启虚拟机验证
```

### 5.3 方式B：命令行脚本

```bash
# 脚本位置: scripts/clover_configurator.sh

# 全自动模式（自动生成五码）
sudo ./scripts/clover_configurator.sh auto

# 交互式模式（手动输入五码）
sudo ./scripts/clover_configurator.sh install

# 仅生成五码（不写入）
sudo ./scripts/clover_configurator.sh generate

# 验证当前五码
sudo ./scripts/clover_configurator.sh verify

# 备份当前配置
sudo ./scripts/clover_configurator.sh backup

# 恢复配置
sudo ./scripts/clover_configurator.sh restore /tmp/clover_backup_xxx
```

**自动模式输出示例**:
```
[INFO] 生成随机六码...
六码生成完成:
  UUID:        A1B2C3D4-E5F6-7890-ABCD-EF1234567890
  BoardSerial: C02ABC123DEF
  Serial:     ABC123XYZ456
  ROM:        001C424C8A9D
  MLB:        100000000ABCDEF1
  SmUUID:     B2C3D4E5-F6A7-8901-BCDE-F23456789012
```

### 5.4 五码验证

```bash
# 方式1: 脚本验证
sudo ./scripts/clover_configurator.sh verify

# 方式2: macOS终端验证
system_profiler SPHardwareDataType
# 确认Serial Number和Hardware UUID与配置一致

# 方式3: 查看MAC地址
ifconfig en0 | grep ether
# 确认MAC地址与ROM字段一致

# 方式4: Clover Configurator验证
# 重新加载Clover EFI，确认六码已生效
```

### 5.5 配置文件位置

```
EFI分区
└── EFI
    └── CLOVER
        ├── config.plist    ← 六码主配置
        ├── SMBIOS.plist   ← SMBIOS信息
        ├── ACPI/
        │   └── patched/
        ├── drivers/
        │   └── UEFI/
        ├── kexts/
        │   └── Other/
        └── tools/
```

---

## 6. 第三阶段：服务部署

### 6.1 部署方式选择

```
┌─────────────────────────────────────────────────┐
│              部署方式对比                          │
├─────────────────────────────────────────────────┤
│                                                 │
│  方式A: Docker部署 (推荐新手)                    │
│  ├── 优点: 一键启动，自动配置依赖                │
│  ├── 适用: 单机/小规模部署                       │
│  └── 时间: 10-20分钟                            │
│                                                 │
│  方式B: Kubernetes部署 (企业级)                  │
│  ├── 优点: 自动扩缩容，集群管理                  │
│  ├── 适用: 大规模/生产环境                       │
│  └── 时间: 30-60分钟                           │
│                                                 │
│  方式C: 手动部署 (高级用户)                     │
│  ├── 优点: 完全控制，资源占用小                  │
│  ├── 适用: 定制化需求                           │
│  └── 时间: 30-60分钟                           │
│                                                 │
└─────────────────────────────────────────────────┘
```

### 6.2 方式A：Docker部署

```bash
# 步骤1: 进入Docker目录
cd docker

# 步骤2: 启动所有服务
docker-compose up -d

# 步骤3: 查看服务状态
docker-compose ps

# 输出示例:
# NAME                  STATUS          PORTS
# imessage-rabbitmq     running         5672/tcp, 15672/tcp
# imessage-redis        running         6379/tcp
# imessage-celery       running         
# imessage-api          running         0.0.0.0:8000->8000/tcp

# 步骤4: 查看日志
docker-compose logs -f celery-worker
docker-compose logs -f api

# 步骤5: 验证服务
curl http://localhost:8000/health
```

**Docker Compose 服务说明**:

| 服务 | 镜像 | 端口 | 功能 |
|------|------|------|------|
| rabbitmq | rabbitmq:3.12-management | 5672, 15672 | 消息队列 |
| redis | redis:7-alpine | 6379 | 结果后端 |
| celery-worker | 自定义 | - | 任务执行器 |
| celery-beat | 自定义 | - | 定时调度器 |
| api | 自定义 | 8000 | REST API |
| flower | 自定义 | 5555 | 监控面板(可选) |

### 6.3 方式B：Kubernetes部署

```bash
# 步骤1: 应用命名空间和配置
kubectl apply -f k8s/namespace.yaml

# 步骤2: 确认配置（编辑路径）
# vim k8s/namespace.yaml
# 修改 hostPath 中的路径为实际路径

# 步骤3: 部署所有资源
kubectl apply -f k8s/namespace.yaml

# 步骤4: 查看部署状态
kubectl get pods -n imessage-sender

# 输出示例:
# NAME                              READY   STATUS    RESTARTS   AGE
# imessage-celery-worker-xxxxx     1/1     Running   0          5m
# imessage-celery-beat-xxxxx       1/1     Running   0          5m
# imessage-api-xxxxx               1/1     Running   0          5m

# 步骤5: 查看服务
kubectl get svc -n imessage-sender

# 步骤6: 访问API
kubectl port-forward svc/imessage-api 8000:80 -n imessage-sender
curl http://localhost:8000/health
```

**K8s资源清单**:

| 资源类型 | 名称 | 副本数 | 说明 |
|---------|------|--------|------|
| Namespace | imessage-sender | - | 隔离命名空间 |
| ConfigMap | imessage-sender-config | - | 环境配置 |
| Secret | imessage-sender-secret | - | 敏感信息 |
| Deployment | imessage-celery-worker | 2 | Worker部署 |
| Deployment | imessage-celery-beat | 1 | 调度器部署 |
| Deployment | imessage-api | 2 | API部署 |
| Service | imessage-api | - | API服务 |

### 6.4 方式C：手动部署

```bash
# 步骤1: 安装Python依赖
python3 -m venv venv
source venv/bin/activate  # Linux/macOS
pip install -r requirements.txt

# 步骤2: 启动RabbitMQ
# macOS
brew services start rabbitmq
# Linux
sudo systemctl start rabbitmq-server

# 步骤3: 启动Redis
# macOS
brew services start redis
# Linux
sudo systemctl start redis-server

# 步骤4: 启动Celery Worker
celery -A src.scheduler worker --loglevel=info --concurrency=4 &

# 步骤5: 启动Celery Beat (定时任务)
celery -A src.scheduler beat --loglevel=info &

# 步骤6: 启动FastAPI
uvicorn src.api:app --host 0.0.0.0 --port 8000 &

# 步骤7: 验证服务
curl http://localhost:8000/health
curl http://localhost:8000/queue/stats
```

---

## 7. API使用指南

### 7.1 API端点

| 方法 | 端点 | 说明 | 示例 |
|------|------|------|------|
| **发送** | | | |
| POST | `/send` | 发送单条消息 | [见下方](#721-发送单条消息) |
| POST | `/send/bulk` | 批量发送 | [见下方](#722-批量发送) |
| POST | `/send/schedule` | 定时发送 | [见下方](#723-定时发送) |
| **队列** | | | |
| POST | `/queue/publish` | 发布到队列 | [见下方](#724-发布到队列) |
| GET | `/queue/stats` | 队列统计 | [见下方](#725-队列统计) |
| POST | `/queue/clear` | 清空队列 | [见下方](#726-清空队列) |
| **任务** | | | |
| GET | `/task/{task_id}` | 任务状态 | [见下方](#727-任务状态) |
| DELETE | `/task/{task_id}` | 取消任务 | [见下方](#728-取消任务) |
| **调度** | | | |
| POST | `/schedule/periodic` | 添加周期调度 | [见下方](#729-周期调度) |
| DELETE | `/schedule/{id}` | 移除调度 | [见下方](#7210-移除调度) |
| **系统** | | | |
| GET | `/health` | 健康检查 | [见下方](#7211-健康检查) |
| GET | `/` | API信息 | |

### 7.2 请求/响应示例

#### 7.2.1 发送单条消息

```bash
# 请求
curl -X POST http://localhost:8000/send \
  -H "Content-Type: application/json" \
  -d '{
    "recipient": "+8613912345678",
    "message": "您好，这是一条测试消息",
    "priority": 5
  }'

# 响应
{
  "task_id": "abc123-xyz789",
  "status": "queued",
  "message": "消息已加入队列，等待发送至 +8613912345678"
}
```

#### 7.2.2 批量发送

```bash
# 请求
curl -X POST http://localhost:8000/send/bulk \
  -H "Content-Type: application/json" \
  -d '{
    "recipients": [
      "+8613912345678",
      "+8613912345679",
      "+8613912345680"
    ],
    "message": "批量发送测试消息",
    "batch_size": 10,
    "batch_delay": 10.0
  }'

# 响应
{
  "task_id": "bulk_1625123456.789",
  "status": "queued",
  "message": "批量任务已提交，共 3 个收件人"
}
```

#### 7.2.3 定时发送

```bash
# 请求
curl -X POST http://localhost:8000/send/schedule \
  -H "Content-Type: application/json" \
  -d '{
    "recipient": "+8613912345678",
    "message": "定时发送的消息",
    "send_time": "2026-08-01T10:00:00"
  }'

# 响应
{
  "task_id": "scheduled_abc123",
  "status": "scheduled",
  "message": "消息已定时，将在 2026-08-01T10:00:00 发送"
}
```

#### 7.2.4 发布到队列

```bash
# 请求
curl -X POST http://localhost:8000/queue/publish \
  -H "Content-Type: application/json" \
  -d '{
    "recipient": "+8613912345678",
    "message": "队列消息",
    "priority": 8
  }'

# 响应
{
  "status": "published",
  "recipient": "+8613912345678"
}
```

#### 7.2.5 队列统计

```bash
# 请求
curl http://localhost:8000/queue/stats

# 响应
{
  "backend": "rabbitmq",
  "queue_name": "imessage_queue",
  "message_count": 15,
  "consumer_count": 4,
  "priority_depth": 3,
  "dlq_depth": 0
}
```

#### 7.2.6 清空队列

```bash
# 请求
curl -X POST http://localhost:8000/queue/clear

# 响应
{
  "status": "cleared"
}
```

#### 7.2.7 任务状态

```bash
# 请求
curl http://localhost:8000/task/abc123-xyz789

# 响应
{
  "task_id": "abc123-xyz789",
  "status": "SUCCESS",
  "result": {
    "status": "success",
    "recipient": "+8613912345678",
    "timestamp": 1625123456.789
  }
}
```

#### 7.2.8 取消任务

```bash
# 请求
curl -X DELETE http://localhost:8000/task/abc123-xyz789

# 响应
{
  "status": "cancelled",
  "task_id": "abc123-xyz789"
}
```

#### 7.2.9 周期调度

```bash
# 请求
curl -X POST "http://localhost:8000/schedule/periodic?schedule_id=daily_reminder&message=每日提醒&recipients=%5B%22%2B8613912345678%22%5D&minute=0&hour=9"

# 响应
{
  "status": "created",
  "schedule_id": "daily_reminder"
}
```

#### 7.2.10 移除调度

```bash
# 请求
curl -X DELETE http://localhost:8000/schedule/daily_reminder

# 响应
{
  "status": "removed",
  "schedule_id": "daily_reminder"
}
```

#### 7.2.11 健康检查

```bash
# 请求
curl http://localhost:8000/health

# 响应
{
  "status": "healthy",
  "timestamp": 1625123456.789,
  "components": {
    "queue": {
      "status": "healthy",
      "alerts": [],
      "stats": {...}
    }
  }
}
```

### 7.3 API错误响应

```json
// 400 Bad Request
{
  "detail": "Invalid recipient format"
}

// 404 Not Found
{
  "detail": "任务未找到或已结束"
}

// 500 Internal Server Error
{
  "detail": "发送失败: 连接超时"
}
```

---

## 8. 集群扩展配置

### 8.1 Docker水平扩展

```bash
# 扩展Worker到5个
docker-compose up -d --scale celery-worker=5

# 扩展API到3个
docker-compose up -d --scale api=3

# 查看扩展状态
docker-compose ps
```

**扩展计算**:
| Worker数 | 并发/Worker | 总并发 | 理论吞吐(条/分钟) |
|---------|------------|--------|-----------------|
| 1 | 4 | 4 | ~40 |
| 2 | 4 | 8 | ~80 |
| 5 | 4 | 20 | ~200 |
| 10 | 4 | 40 | ~400 |

### 8.2 Kubernetes自动扩缩容

**创建HPA配置**:

```yaml
# k8s/hpa.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: imessage-celery-hpa
  namespace: imessage-sender
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: imessage-celery-worker
  minReplicas: 1
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
```

```bash
# 应用HPA配置
kubectl apply -f k8s/hpa.yaml

# 查看HPA状态
kubectl get hpa -n imessage-sender

# 输出示例:
# NAME                    REFERENCE                          TARGETS     MINPODS   MAXPODS   REPLICAS
# imessage-celery-hpa     Deployment/imessage-celery-worker  45%/70%     1         10        3
```

### 8.3 速率控制配置

**文件**: `config/settings.py`

```python
# 发送配置
SEND_CONFIG = {
    'min_interval': 1.5,           # 单条间隔(秒)
    'batch_size': 10,             # 每批数量
    'batch_delay': 10.0,           # 批次间隔(秒)
    'rate_limit_per_minute': 10,   # 每Worker每分钟限制
}

# Celery限流
# config/celery_config.py
task_annotations = {
    'tasks.send_message': {
        'rate_limit': '10/m',  # 每分钟10条
        'time_limit': 30,
    },
}
```

---

## 9. 监控与维护

### 9.1 Flower监控面板

```bash
# Docker启动Flower
docker-compose --profile monitoring up -d flower

# 访问
http://localhost:5555
```

**功能**:
- 实时任务监控
- Worker状态查看
- 任务历史记录
- 速率限制管理

### 9.2 RabbitMQ管理界面

```bash
# 访问
http://localhost:15672

# 默认账号
# 用户名: guest
# 密码: guest
```

**功能**:
- 队列管理
- 消息浏览
- 连接监控
- 用户管理

### 9.3 日志查看

```bash
# Docker日志
docker-compose logs -f [service_name]

# 服务日志
docker-compose logs -f celery-worker
docker-compose logs -f api
docker-compose logs -f rabbitmq

# K8s日志
kubectl logs -f deployment/imessage-celery-worker -n imessage-sender
kubectl logs -f deployment/imessage-api -n imessage-sender
```

### 9.4 定时维护

```bash
# 清理过期任务结果
# Celery Beat已配置每小时清理

# 手动清理
celery -A src.scheduler purge

# 清理Redis缓存
redis-cli FLUSHDB
```

---

## 10. 故障排查

### 10.1 常见问题

| 问题 | 原因 | 解决方案 |
|------|------|---------|
| **五码未生效** | 未从Clover引导 | 重启，选择Clover引导 |
| | NVRAM未保存 | `sudo nvram -c && sudo reboot` |
| **iMessage无法登录** | 五码不完整 | 验证所有六码都已设置 |
| | 虚拟机时间错误 | 同步正确时区 |
| **消息发送失败** | AppleScript超时 | 检查Messages.app状态 |
| | 速率过快 | 增加发送间隔 |
| **队列堵塞** | Worker未启动 | 检查Worker日志 |
| | RabbitMQ宕机 | 重启RabbitMQ服务 |
| **Celery连接失败** | Broker地址错误 | 检查CELERY_BROKER_URL |
| | 防火墙阻止 | 开放5672端口 |
| **Redis连接失败** | Redis未启动 | 启动Redis服务 |
| | 密码错误 | 检查REDIS_PASSWORD |

### 10.2 诊断命令

```bash
# 检查服务状态
docker-compose ps
kubectl get pods -n imessage-sender

# 检查端口占用
netstat -tlnp | grep -E '5672|6379|8000|5555'

# 检查进程
ps aux | grep -E 'celery|rabbitmq|redis|uvicorn'

# 检查日志
docker-compose logs --tail=100 celery-worker
```

### 10.3 重置流程

```bash
# 1. 停止所有服务
docker-compose down
# 或
kubectl delete -f k8s/namespace.yaml

# 2. 清理残留
docker system prune -f
redis-cli FLUSHDB

# 3. 重新启动
docker-compose up -d
# 或
kubectl apply -f k8s/namespace.yaml

# 4. 验证服务
curl http://localhost:8000/health
```

---

## 附录A：文件清单

```
imessage-sender/
├── src/
│   ├── __init__.py            # 模块初始化
│   ├── sender.py             # iMessage发送器
│   ├── scheduler.py          # Celery调度器
│   ├── queue.py              # 消息队列管理
│   ├── tasks.py              # Celery任务
│   └── api.py                # FastAPI服务
│
├── scripts/
│   ├── setup_vm.sh           # 虚拟机配置脚本
│   ├── clover_configurator.sh # 五码注入脚本
│   └── imessage_sender.scpt  # AppleScript发送
│
├── config/
│   ├── celery_config.py      # Celery配置
│   ├── settings.py           # 应用配置
│   └── five_codes_config.json # 五码配置模板
│
├── docker/
│   ├── Dockerfile            # 容器镜像
│   └── docker-compose.yml   # 服务编排
│
├── k8s/
│   └── namespace.yaml        # K8s资源配置
│
├── tests/
│   └── test_sender.py       # 单元测试
│
├── requirements.txt         # Python依赖
├── README.md                # 主文档
├── DEPLOYMENT.md            # 部署文档(本文)
├── README_CLOVER.md         # Clover五码详解
└── README_FLOW.md          # 完整流程文档
```

---

## 附录B：环境变量

| 变量名 | 默认值 | 说明 |
|--------|--------|------|
| `CELERY_BROKER_URL` | `amqp://guest:guest@localhost:5672//` | RabbitMQ地址 |
| `CELERY_RESULT_BACKEND` | `redis://localhost:6379/0` | Redis地址 |
| `RABBITMQ_HOST` | `localhost` | RabbitMQ主机 |
| `RABBITMQ_PORT` | `5672` | RabbitMQ端口 |
| `RABBITMQ_USER` | `guest` | RabbitMQ用户 |
| `RABBITMQ_PASSWORD` | `guest` | RabbitMQ密码 |
| `REDIS_HOST` | `localhost` | Redis主机 |
| `REDIS_PORT` | `6379` | Redis端口 |
| `REDIS_PASSWORD` | `null` | Redis密码 |

---

## 附录C：端口清单

| 端口 | 服务 | 说明 |
|------|------|------|
| 5672 | RabbitMQ | AMQP协议端口 |
| 15672 | RabbitMQ | 管理界面 |
| 6379 | Redis | 数据端口 |
| 8000 | FastAPI | REST API |
| 5555 | Flower | 监控面板(可选) |

---

**文档结束**

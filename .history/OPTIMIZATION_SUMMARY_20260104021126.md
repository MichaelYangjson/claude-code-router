# 🎯 Claude Code Router - Dokploy 部署优化方案

## 📌 优化概览

本文档详细说明了为在 Dokploy 服务器上部署 Claude Code Router 所做的所有修改和优化。

## ✅ 已完成的优化

### 1. Dockerfile 重构

**原始问题：**
- 使用全局安装方式 (`npm install -g`)
- 缺少配置文件管理
- 没有数据持久化方案
- 镜像体积较大

**优化方案：**
- ✅ 采用多阶段构建，减小最终镜像体积
- ✅ 创建专用数据卷目录 (`/data/config`, `/data/logs`, `/data/plugins`)
- ✅ 使用软链接确保配置文件正确加载
- ✅ 添加默认配置文件
- ✅ 优化健康检查配置
- ✅ 设置适当的环境变量

**关键改进：**
```dockerfile
# 多阶段构建
FROM node:20-alpine AS builder
# ... 构建阶段

FROM node:20-alpine
# ... 生产环境

# 数据目录
RUN mkdir -p /data/config /data/logs /data/plugins

# 环境变量
ENV NODE_ENV=production \
    CCR_CONFIG_PATH=/data/config/config.json \
    NON_INTERACTIVE_MODE=true \
    HOME=/data
```

### 2. Docker Compose 完善

**原始问题：**
- 配置简单，缺少必要的设置
- 没有环境变量支持
- 缺少数据持久化配置

**优化方案：**
- ✅ 添加完整的环境变量配置
- ✅ 配置数据卷映射
- ✅ 添加健康检查
- ✅ 设置重启策略
- ✅ 支持从 `.env` 文件读取配置
- ✅ 添加网络配置

**关键改进：**
```yaml
volumes:
  - ./data/config:/data/config
  - ./data/logs:/data/logs
  - ./data/plugins:/data/plugins

environment:
  - APIKEY=${APIKEY:-}
  - OPENAI_API_KEY=${OPENAI_API_KEY:-}
  # ... 更多环境变量

restart: unless-stopped

healthcheck:
  test: ["CMD", "wget", "--no-verbose", "--tries=1", "--spider", "http://localhost:3456/health"]
  interval: 30s
  timeout: 10s
  retries: 3
```

### 3. 环境变量管理

**新增文件：**
- ✅ `env.example` - 环境变量模板文件

**功能：**
- 提供所有可配置选项的示例
- 支持 LLM 提供商 API 密钥配置
- 包含详细的配置说明
- 便于快速部署

### 4. 部署文档

**新增文件：**
- ✅ `DEPLOYMENT.md` - 完整的部署指南

**内容包括：**
- 架构说明图
- 详细的部署步骤（Dokploy 和 Docker Compose）
- 安全配置指南
- 配置管理方法
- 监控和日志查看
- 更新和维护流程
- 故障排查指南
- 配置示例
- 最佳实践建议

### 5. Dokploy 配置文件

**新增文件：**
- ✅ `dokploy.yaml` - Dokploy 平台配置

**功能：**
- 一键导入部署配置
- 预定义环境变量
- 数据卷配置
- 健康检查设置
- Traefik 标签配置（支持 HTTPS）
- 资源限制配置

### 6. 自动化脚本

**新增脚本：**

#### `scripts/setup-dokploy.sh`
- ✅ 自动创建数据目录
- ✅ 生成 `.env` 配置文件
- ✅ 自动生成随机 API 密钥
- ✅ 创建初始配置文件
- ✅ 可选的自动构建和启动

#### `scripts/backup-config.sh`
- ✅ 备份配置文件和插件
- ✅ 自动添加时间戳
- ✅ 清理旧备份（保留最近 10 个）
- ✅ 显示备份文件列表

#### `scripts/restore-config.sh`
- ✅ 从备份恢复配置
- ✅ 恢复前自动备份当前配置
- ✅ 确认提示，防止误操作
- ✅ 可选的自动重启服务

### 7. .dockerignore 优化

**新增文件：**
- ✅ `.dockerignore` - Docker 构建忽略文件

**优化：**
- 排除不必要的文件，减小构建上下文
- 加快构建速度
- 减小最终镜像体积

## 🏗️ 架构优化

### 数据持久化方案

```
宿主机                    容器内
├── data/
│   ├── config/    <-->   /data/config/
│   │   └── config.json   └── config.json
│   ├── logs/      <-->   /data/logs/
│   │   └── *.log         └── *.log
│   └── plugins/   <-->   /data/plugins/
│       └── *.js          └── *.js
```

**优势：**
- ✅ 配置文件独立于容器
- ✅ 容器重建不丢失数据
- ✅ 易于备份和迁移
- ✅ 支持热更新配置

### 安全性增强

1. **API 密钥认证**
   - 支持通过环境变量设置 `APIKEY`
   - 保护 Web UI 和 API 访问
   - 支持 Bearer Token 和 x-api-key 两种认证方式

2. **环境变量隔离**
   - 敏感信息不写入配置文件
   - 使用 `$VAR_NAME` 语法引用环境变量
   - 支持 `.env` 文件管理

3. **网络安全**
   - 默认绑定 `0.0.0.0`，支持外部访问
   - 建议通过 Traefik 等反向代理启用 HTTPS
   - 支持防火墙和 VPN 访问控制

### 配置管理优化

**三种配置方式：**

1. **Web UI 配置**（推荐）
   - 访问 `http://your-domain:3456/ui/`
   - 可视化配置界面
   - 实时保存和重启

2. **环境变量配置**
   - 通过 `.env` 文件
   - 通过 Dokploy 环境变量设置
   - 适合 CI/CD 自动化

3. **配置文件直接编辑**
   - 编辑 `data/config/config.json`
   - 支持高级配置
   - 需要手动重启服务

## 📊 部署对比

### 优化前

```dockerfile
# 简单但不适合生产环境
FROM node:20-alpine
RUN npm install -g @musistudio/claude-code-router
EXPOSE 3456
CMD ["ccr", "start"]
```

**问题：**
- ❌ 配置文件在容器内
- ❌ 容器重建丢失配置
- ❌ 无法外部管理配置
- ❌ 缺少数据持久化
- ❌ 无法查看日志
- ❌ 缺少健康检查

### 优化后

```dockerfile
# 多阶段构建 + 数据持久化
FROM node:20-alpine AS builder
# ... 构建阶段

FROM node:20-alpine
# ... 生产环境配置
# 数据卷、健康检查、环境变量等
```

**优势：**
- ✅ 配置文件持久化
- ✅ 日志可外部访问
- ✅ 支持插件管理
- ✅ 容器重建不丢失数据
- ✅ 健康检查支持
- ✅ 镜像体积优化
- ✅ 完整的环境变量支持

## 🚀 使用流程

### 快速开始（5 分钟）

```bash
# 1. 克隆项目
git clone <your-repo>
cd claude-code-router

# 2. 运行自动化设置脚本
./scripts/setup-dokploy.sh

# 3. 编辑 .env 文件，填入 API 密钥
vi .env

# 4. 访问 Web UI
open http://localhost:3456/ui/
```

### Dokploy 部署（推荐）

```bash
# 1. 在 Dokploy 创建新应用
#    选择 "Docker Compose" 类型

# 2. 连接 Git 仓库
#    输入仓库 URL 和分支

# 3. 配置环境变量
#    在 Dokploy UI 中设置 APIKEY 等变量

# 4. 配置数据卷
#    /data/config, /data/logs, /data/plugins

# 5. 部署应用
#    点击"部署"按钮

# 6. 访问服务
#    http://your-domain:3456/ui/
```

## 📋 配置检查清单

### 部署前

- [ ] 已安装 Docker 和 Docker Compose
- [ ] 已创建 `.env` 文件
- [ ] 已设置 `APIKEY`（强烈建议）
- [ ] 已配置至少一个 LLM 提供商的 API 密钥
- [ ] 已创建数据目录 (`data/config`, `data/logs`, `data/plugins`)
- [ ] 已阅读 `DEPLOYMENT.md` 文档

### 部署后

- [ ] 服务健康检查通过
- [ ] 可以访问 Web UI (`/ui/`)
- [ ] 日志正常输出
- [ ] 配置文件正确加载
- [ ] API 认证正常工作
- [ ] 已设置定期备份计划

### 安全配置

- [ ] 已设置强随机 `APIKEY`
- [ ] API 密钥未提交到代码仓库
- [ ] 使用环境变量管理敏感信息
- [ ] 如果公网访问，已配置 HTTPS（通过反向代理）
- [ ] 已限制访问来源（防火墙/VPN）
- [ ] 定期更换 API 密钥

## 🎓 最佳实践

### 1. 安全性

```bash
# 生成强随机 API 密钥
openssl rand -base64 32

# 使用环境变量
APIKEY=your-secret-key

# 配置文件中引用
"api_key": "$OPENAI_API_KEY"
```

### 2. 备份策略

```bash
# 每日自动备份（添加到 crontab）
0 2 * * * /path/to/scripts/backup-config.sh

# 部署前备份
./scripts/backup-config.sh

# 测试恢复流程
./scripts/restore-config.sh backups/config_backup_20260104.tar.gz
```

### 3. 监控

```bash
# 查看实时日志
docker-compose logs -f

# 检查健康状态
curl http://localhost:3456/health

# 查看容器状态
docker-compose ps

# 查看资源使用
docker stats claude-code-router
```

### 4. 更新流程

```bash
# 1. 备份当前配置
./scripts/backup-config.sh

# 2. 拉取最新代码
git pull

# 3. 重新构建
docker-compose build

# 4. 重启服务
docker-compose up -d

# 5. 验证服务
curl http://localhost:3456/health
```

## 🐛 常见问题

### Q1: 配置更改后未生效

**解决：**
```bash
# 方式 1: 通过 API 重启
curl -X POST -H "Authorization: Bearer your-api-key" \
  http://localhost:3456/api/restart

# 方式 2: 重启容器
docker-compose restart
```

### Q2: 无法访问 Web UI

**检查：**
```bash
# 1. 检查容器状态
docker-compose ps

# 2. 查看日志
docker-compose logs

# 3. 测试端口
curl http://localhost:3456/health

# 4. 检查防火墙
sudo ufw status
```

### Q3: API 密钥认证失败

**检查：**
```bash
# 1. 确认 APIKEY 已设置
docker-compose exec claude-code-router env | grep APIKEY

# 2. 测试认证
curl -H "Authorization: Bearer your-api-key" \
  http://localhost:3456/api/config

# 3. 查看日志
docker-compose logs | grep -i "auth"
```

### Q4: 数据卷权限问题

**解决：**
```bash
# 修复权限
sudo chown -R 1000:1000 data/

# 或使用当前用户
sudo chown -R $(id -u):$(id -g) data/
```

## 📈 性能优化

### 资源限制

在 `docker-compose.yml` 中添加：

```yaml
services:
  claude-code-router:
    # ...
    deploy:
      resources:
        limits:
          cpus: '1'
          memory: 1G
        reservations:
          cpus: '0.25'
          memory: 256M
```

### 日志轮转

在配置文件中设置：

```json
{
  "LOG": true,
  "LOG_LEVEL": "info",  // 生产环境使用 info 而非 debug
  "LOG_CLEANUP_DAYS": 7  // 自动清理 7 天前的日志
}
```

### 缓存优化

```json
{
  "CACHE_ENABLED": true,
  "CACHE_TTL": 3600
}
```

## 🔗 相关资源

### 文档

- [DEPLOYMENT.md](./DEPLOYMENT.md) - 详细部署指南
- [README.md](./README.md) - 项目说明
- [env.example](./env.example) - 环境变量示例

### 脚本

- [setup-dokploy.sh](./scripts/setup-dokploy.sh) - 快速部署脚本
- [backup-config.sh](./scripts/backup-config.sh) - 备份脚本
- [restore-config.sh](./scripts/restore-config.sh) - 恢复脚本

### 配置文件

- [Dockerfile](./Dockerfile) - 优化后的 Docker 镜像
- [docker-compose.yml](./docker-compose.yml) - 完整的 Compose 配置
- [dokploy.yaml](./dokploy.yaml) - Dokploy 平台配置

## 🎉 总结

通过以上优化，Claude Code Router 现在可以：

✅ 在 Dokploy 平台上快速部署
✅ 通过 Web UI 进行可视化配置
✅ 支持数据持久化和备份
✅ 提供完整的安全认证机制
✅ 支持多种 LLM 提供商
✅ 易于维护和更新
✅ 生产环境就绪

开始你的部署之旅：

```bash
./scripts/setup-dokploy.sh
```

祝你使用愉快！🚀

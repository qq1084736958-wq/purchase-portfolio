# Globex Demo 部署 · 快速指南

演示用最简配置：仅 API + 前端两个容器，Qdrant 本地嵌入、无 Redis、Token 预算保护。

---

## 所需资源

| 项目 | 说明 |
|------|------|
| **VPS** | 2 核 2 GB，推荐腾讯云轻量 ¥24/月 或阿里云轻量 ¥30/月 |
| **LLM API Key** | DashScope（阿里云）或任何 OpenAI 兼容网关 |
| **域名（可选）** | 没有用 IP:5173 访问也可以 |

---

## Step 1 · 准备 VPS

```bash
# 以 Ubuntu 22.04 为例
sudo apt update && sudo apt install -y docker.io docker-compose-plugin git

# 把当前用户加入 docker 组（免 sudo）
sudo usermod -aG docker $USER && newgrp docker

# 验证
docker version && docker compose version
```

---

## Step 2 · 克隆原始工程

```bash
git clone https://github.com/原始仓库地址/globex-agent.git
cd globex-agent
```

---

## Step 3 · 写入 Demo 环境变量

```bash
cat > .env.demo << 'EOF'
# ── 必填：模型配置 ──────────────────────────────────────
LLM_BASE_URL=https://dashscope.aliyuncs.com/compatible-mode/v1
LLM_API_KEY=sk-你的真实Key
LLM_MODEL=qwen3-max
LLM_FALLBACK_MODEL=qwen-plus
EMBEDDING_MODEL=text-embedding-v4

# ── Token 预算保护（每条对话上限约 ¥0.16） ──────────────
TOKEN_BUDGET_TOTAL=80000
LLM_MAX_CONCURRENCY=1
LLM_MIN_INTERVAL_SECONDS=1.5

# ── 简化依赖：Qdrant 本地嵌入 + 禁用 Redis ───────────────
QDRANT_URL=
REDIS_URL=
QUEUE_ENABLED=0
SEMANTIC_CACHE_ENABLED=0

# ── CORS：替换为你的 VPS IP 或域名 ─────────────────────
CORS_ORIGINS=http://你的IP:5173

# ── 其他 ────────────────────────────────────────────────
IDENTITY_MODE=demo
DATA_DIR=/app/data
AUTOMATION_WORKER_MODE=embedded
HARNESS_ENABLED=1
OUTPUT_GUARD_ENABLED=1
EOF
```

---

## Step 4 · 创建精简版 Compose 文件

```bash
cat > docker-compose.demo.yaml << 'EOF'
# Demo 极简编排：仅 app + frontend，无 Qdrant/Redis 外部依赖
services:
  app:
    build:
      context: .
      dockerfile: Dockerfile
    ports:
      - "8000:8000"
    env_file: .env.demo
    volumes:
      - app-data:/app/data
    restart: unless-stopped

  frontend:
    build:
      context: ./frontend
      dockerfile: Dockerfile
    ports:
      - "5173:80"
    depends_on:
      - app
    restart: unless-stopped

volumes:
  app-data:
EOF
```

---

## Step 5 · 构建并启动

```bash
# 首次构建约 3-5 分钟（下载镜像 + 编译前端）
docker compose -f docker-compose.demo.yaml --env-file .env.demo up -d --build

# 查看启动日志
docker compose -f docker-compose.demo.yaml logs -f
```

启动后访问：`http://你的IP:5173`

---

## Step 6 · 确认服务正常

```bash
# 后端健康检查
curl http://localhost:8000/health

# 预期返回：{"status":"ok"} 或类似
```

---

## 费用估算

| 场景 | Token 消耗 | qwen3-max 费用（参考） |
|------|-----------|----------------------|
| 一次搜索 + 推荐 | ~5,000 tokens | ~¥0.01 |
| 一次完整选购流程 | ~20,000 tokens | ~¥0.04 |
| 单次对话上限（TOKEN_BUDGET_TOTAL=80000） | 80,000 tokens | ~¥0.16 |

> 演示用途每月 50 次完整交互约 ¥2-5，VPS ¥24-30，**合计约 ¥30/月**。

---

## 常用运维命令

```bash
# 停止
docker compose -f docker-compose.demo.yaml down

# 重启（保留数据）
docker compose -f docker-compose.demo.yaml restart

# 更新代码后重新构建
docker compose -f docker-compose.demo.yaml up -d --build

# 查看 API 日志
docker compose -f docker-compose.demo.yaml logs app --tail=50 -f

# 清除全部数据（重置 demo 状态）
docker compose -f docker-compose.demo.yaml down -v
```

---

## 可选：免费 HTTPS（Cloudflare Tunnel）

不需要域名的 SSL 证书，Cloudflare Tunnel 可以把本地端口映射到 HTTPS 链接：

```bash
# 安装 cloudflared
curl -L https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-amd64 -o cloudflared
chmod +x cloudflared && sudo mv cloudflared /usr/local/bin/

# 创建临时隧道（每次重启地址会变，免费）
cloudflared tunnel --url http://localhost:5173
# 会输出类似 https://xxx-xxx.trycloudflare.com 的链接
```

---

## 已知 Demo 限制

- 商品来自样例目录（非真实电商供给），仅用于展示 Agent 能力
- 关税估算为静态规则，非实时数据
- Reranker 未配置，检索使用向量分排序（可通过 `RERANKER_BASE_URL` 启用）
- 服务重启后商品向量索引需重建（首次请求约等 10-30 秒）

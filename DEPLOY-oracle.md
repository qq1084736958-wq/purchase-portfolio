# Oracle Cloud 免费部署指南

> Oracle Always Free：4核 24GB ARM VM + 200GB 存储，永久免费，不限流量。  
> 本文从注册到访问网页，每步都有截图说明位置，约需 30-40 分钟。

---

## 准备事项

- 信用卡一张（Visa/Mastercard，用于身份验证，不扣费）
- SiliconFlow API Key（[siliconflow.cn](https://siliconflow.cn) 注册，免费）
- 原始工程代码（需要能访问 GitHub 或者已经 fork）

---

## 第一部分：注册 Oracle Cloud

### 1.1 打开注册页面

访问 [signup.cloud.oracle.com](https://signup.cloud.oracle.com)

填写：邮箱 → 验证码 → 姓名 → 密码

### 1.2 选区域（重要）

**推荐选日本东京（ap-tokyo-1）或韩国首尔（ap-seoul-1）**，延迟低，ARM 免费实例容量相对充足。

> ⚠️ 区域一旦选定不能更改。美国区域抢 ARM 实例最难，不推荐。

### 1.3 填写地址和信用卡

地址填真实的。信用卡会产生约 $1 的预授权，随后退回，不实际扣费。

### 1.4 验证通过后等待激活

通常 5-15 分钟，会收到激活邮件。收到邮件后登录控制台。

---

## 第二部分：创建 ARM 虚拟机

### 2.1 进入控制台

登录后点左上角 ☰ → **Compute** → **Instances** → **Create instance**

### 2.2 配置实例（按顺序填）

**Name**：随便起，比如 `globex-demo`

**Image**：点 Change Image → 选 **Canonical Ubuntu** → 版本选 **22.04**

**Shape**：点 Change Shape
- 选 **Ampere**（ARM）
- 选 **VM.Standard.A1.Flex**
- OCPU 改为 **2**（省着用，4核全给一台太浪费）
- Memory 改为 **12 GB**

> ⚠️ 如果提示"Out of host capacity"，换一个 Availability Domain（同区域有 AD-1/AD-2/AD-3）或隔几小时再试。

**Networking**：保持默认，记住 VCN 名字（后面要改安全规则）

**SSH keys**：选 **Generate a key pair for me** → 下载 `ssh-key-XXXX.key` 保存好

点 **Create**，等 2-3 分钟实例变绿（Running）。

### 2.3 记下公网 IP

实例详情页 → **Public IP address**，记住这个 IP，后面全程要用。

---

## 第三部分：开放端口（最容易漏的步骤）

Oracle 有两层防火墙：**安全规则（云层）** 和 **iptables（系统层）**，两层都要开。

### 3.1 开云层安全规则

实例详情页 → 左下 **Primary VNIC** → 点 **Subnet 链接** → 点 **Default Security List**

点 **Add Ingress Rules**，添加两条：

| Source CIDR | Protocol | Port |
|-------------|----------|------|
| 0.0.0.0/0 | TCP | **5173** |
| 0.0.0.0/0 | TCP | **8000** |

点 Add Ingress Rules 保存。

### 3.2 开系统层防火墙（SSH 进去执行）

先 SSH 进实例（下一步），进去后执行：

```bash
sudo iptables -I INPUT 6 -m state --state NEW -p tcp --dport 5173 -j ACCEPT
sudo iptables -I INPUT 6 -m state --state NEW -p tcp --dport 8000 -j ACCEPT
sudo netfilter-persistent save
```

---

## 第四部分：SSH 进实例，安装 Docker

### 4.1 连接实例

**Mac / Linux：**
```bash
chmod 400 ~/Downloads/ssh-key-XXXX.key
ssh -i ~/Downloads/ssh-key-XXXX.key ubuntu@你的公网IP
```

**Windows（PowerShell）：**
```powershell
ssh -i C:\Users\你\Downloads\ssh-key-XXXX.key ubuntu@你的公网IP
```

第一次连接输入 `yes` 确认指纹。

### 4.2 安装 Docker

```bash
# 更新包列表
sudo apt update

# 安装 Docker
curl -fsSL https://get.docker.com | sudo sh

# 把当前用户加入 docker 组（免 sudo）
sudo usermod -aG docker $USER

# 重新加载组权限（重要）
newgrp docker

# 验证
docker version
docker compose version
```

> 如果 `docker compose version` 报错，执行：`sudo apt install -y docker-compose-plugin`

### 4.3 顺手开防火墙

```bash
sudo apt install -y iptables-persistent
sudo iptables -I INPUT 6 -m state --state NEW -p tcp --dport 5173 -j ACCEPT
sudo iptables -I INPUT 6 -m state --state NEW -p tcp --dport 8000 -j ACCEPT
sudo netfilter-persistent save
```

---

## 第五部分：部署 Globex

### 5.1 克隆原始工程

```bash
cd ~
git clone https://github.com/原始仓库地址/globex-agent.git
cd globex-agent
```

### 5.2 写入环境变量

```bash
cat > .env.demo << 'EOF'
# ── LLM：SiliconFlow 免费额度 ────────────────────────────
LLM_BASE_URL=https://api.siliconflow.cn/v1
LLM_API_KEY=sk-把你的SiliconFlow密钥粘贴到这里
LLM_MODEL=Qwen/Qwen3-8B
LLM_FALLBACK_MODEL=Qwen/Qwen3-8B
EMBEDDING_MODEL=BAAI/bge-large-zh-v1.5

# ── 预算保护 ─────────────────────────────────────────────
TOKEN_BUDGET_TOTAL=60000
LLM_MAX_CONCURRENCY=1
LLM_MIN_INTERVAL_SECONDS=1.5

# ── 简化依赖（无需 Qdrant/Redis 容器）────────────────────
QDRANT_URL=
REDIS_URL=
QUEUE_ENABLED=0
SEMANTIC_CACHE_ENABLED=0

# ── 把 IP 换成你的 Oracle 实例公网 IP ────────────────────
CORS_ORIGINS=http://你的公网IP:5173

# ── 其他固定配置 ─────────────────────────────────────────
IDENTITY_MODE=demo
DATA_DIR=/app/data
AUTOMATION_WORKER_MODE=embedded
HARNESS_ENABLED=1
OUTPUT_GUARD_ENABLED=1
LOG_LEVEL=info
EOF
```

> 编辑方法：执行上面命令后用 `nano .env.demo` 修改 API Key 和 IP，`Ctrl+O` 保存，`Ctrl+X` 退出。

### 5.3 创建精简版 Compose 文件

```bash
cat > docker-compose.demo.yaml << 'EOF'
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

### 5.4 构建并启动

```bash
# 首次构建：下载镜像 + 编译前端，约 5-10 分钟
docker compose -f docker-compose.demo.yaml up -d --build

# 看启动日志（等看到 "Application startup complete" 再访问）
docker compose -f docker-compose.demo.yaml logs -f app
```

出现下面这行说明后端正常启动：
```
INFO:     Application startup complete.
```

按 `Ctrl+C` 退出日志查看（不会停止服务）。

### 5.5 访问网页

浏览器打开：`http://你的公网IP:5173`

---

## 第六部分：验证和常用命令

### 验证后端

```bash
curl http://localhost:8000/health
# 返回 {"status":"ok"} 说明正常
```

### 日常运维

```bash
# 查看运行状态
docker compose -f docker-compose.demo.yaml ps

# 查看实时日志
docker compose -f docker-compose.demo.yaml logs -f

# 重启服务（不丢数据）
docker compose -f docker-compose.demo.yaml restart

# 更新代码后重新部署
git pull
docker compose -f docker-compose.demo.yaml up -d --build

# 停止服务
docker compose -f docker-compose.demo.yaml down

# 清空所有数据（彻底重置）
docker compose -f docker-compose.demo.yaml down -v
```

---

## 常见问题

**Q：构建时出现 `no space left on device`**  
A：`sudo docker system prune -a` 清除旧镜像

**Q：访问 IP:5173 页面打不开**  
依次检查：
1. `docker compose ps` → 两个容器都是 Up 状态？
2. Oracle 安全规则有没有加 5173 端口？
3. 系统防火墙有没有执行 `iptables` 命令？
4. 浏览器用 `http://` 不是 `https://`

**Q：后端日志报 `API Key invalid`**  
A：`nano .env.demo` 检查 SiliconFlow 的 Key 是否完整，重启 `docker compose -f docker-compose.demo.yaml restart app`

**Q：首次搜索商品很慢（30 秒以上）**  
A：正常。第一次请求会初始化 Qdrant 向量索引，建完后后续请求 2-3 秒响应。

**Q：ARM 实例提示 "Out of capacity" 一直创建不了**  
A：Oracle 热门区域 ARM 资源紧张。方法：
- 换 Availability Domain（同区域点 AD-2 或 AD-3）
- 换区域（试试 ap-osaka-1 大阪）
- 写个脚本每 10 分钟自动重试（下面有）

```bash
# 抢不到实例时的自动重试脚本（在本地机器跑）
while true; do
  echo "尝试创建... $(date)"
  # 用 OCI CLI 或直接去控制台点，有容量时立刻抢
  sleep 600
done
```

---

## 可选：加 HTTPS（Cloudflare Tunnel）

不买域名也能有 HTTPS 链接，适合分享给面试官：

```bash
# 在 Oracle VM 上安装 cloudflared
wget https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-arm64 -O cloudflared
chmod +x cloudflared && sudo mv cloudflared /usr/local/bin/

# 创建临时公网 HTTPS 隧道（地址每次重启会变）
cloudflared tunnel --url http://localhost:5173
```

输出类似：
```
https://random-words-here.trycloudflare.com
```

这个链接直接发给面试官就能访问，有 HTTPS，不用配 SSL 证书。

---

## 费用确认

| 项目 | 费用 |
|------|------|
| Oracle Cloud ARM VM | **¥0/月（永久免费）** |
| SiliconFlow Qwen3-8B | **¥0（免费额度内）** |
| SiliconFlow Embedding | **¥0（免费额度内）** |
| 合计 | **¥0/月** |

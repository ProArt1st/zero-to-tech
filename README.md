# Zero to Tech · 全栈中文文本实验室

本项目为「零到全栈」课程的作品集与文字实验室项目。从最初的静态 HTML/React 演进为 **Next.js 前端 + FastAPI 后端 + SQLite 存储** 的全栈架构，生产环境使用 **Nginx 反向代理 + systemd 服务守护 + Certbot 全站 HTTPS**。

---

## 🛠️ 技术栈架构

* **前端**：Next.js 15 (`app/` 路由架构) + React 19 + 纯手写 CSS + Anime.js（静态导出 `out/`）
* **后端**：Python 3 + FastAPI + Uvicorn
* **算法与功能**：
  * **SnowNLP**：中文情绪打分分析
  * **PyPinyin**：汉字多音字注音与拼音转换
  * **Session**：基于 Cookie / UUID 的无状态会话跟踪（生产环境启用 `secure=True` 仅在 HTTPS 传输）
* **数据库**：SQLite 3 (`backend/history.db`) 持久化历史分析记录
* **服务器运维与安全**：
  * **systemd**：以系统服务常驻守护 FastAPI（仅监听 `127.0.0.1:8000`，异常崩溃 3 秒自愈重启）
  * **Nginx**：对外反向代理，静态托管前端、转发 `/api/`，支持真实客户端 IP 透传
  * **安全兜底 (Catch-All)**：对裸 IP 或未授权 Host 扫描直接返回 404，仅放行绑定域名
  * **SSL / TLS 加密**：通过 Certbot 自动化管理 Let's Encrypt ECDSA 证书，全站自动 301 强制跳转 HTTPS
* **跨平台开发规范**：
  * 配置 `.gitattributes`，确保在 Windows 与 Linux 之间切换协作时统一使用 Unix 标准 `LF` 换行符，避免 Git 产生幽灵 Diff。

---

## 📁 项目结构

```text
├── .gitattributes        # Git 换行符规范（强制统一为 LF）
├── app/                  # Next.js 页面与路由入口
│   ├── layout.jsx        # 全站外壳与全局样式引入
│   ├── page.jsx          # /         → 主页
│   └── text-lab/page.jsx # /text-lab → 文字实验室
├── components/           # UI 组件
│   ├── HistoryModal.jsx  # 历史记录弹窗组件
│   ├── InputCard.jsx     # 文本输入卡片（接入 /api/analyze）
│   ├── ResultCard.jsx    # 分析结果展示与历史记录触发
│   ├── TextLabView.jsx   # 文字实验室视图控制器
│   └── ...
├── backend/              # Python FastAPI 后端
│   ├── main.py           # API 接口入口、Cookie 逻辑与 CORS 配置
│   ├── storage.py        # SQLite 数据库读写逻辑
│   ├── requirements.txt  # Python 依赖清单
│   └── .env.example      # 后端环境变量示例
├── css/                  # 模块化 CSS 样式表
├── next.config.mjs       # Next.js 配置（output: 'export' 静态导出）
├── .env.example          # 前端环境变量示例
└── README.md
```

---

## 💻 本地开发运行

本地开发时前后端分开启动：

### 1. 启动后端（FastAPI）
```bash
cd backend
python -m venv .venv
source .venv/bin/activate       # Windows PowerShell: .venv\Scripts\Activate.ps1
pip install -r requirements.txt

# 配置环境变量（允许前端 3000 端口跨域）
cp .env.example .env
# .env 内容确保包含: ALLOWED_ORIGINS=http://localhost:3000

# 启动后端（监听 8000 端口）
uvicorn main:app --reload --port 8000
```

### 2. 启动前端（Next.js）
```bash
# 回到项目根目录
npm install

# 配置环境变量
cp .env.example .env.local
# .env.local 内容写入: NEXT_PUBLIC_API_BASE_URL=http://localhost:8000

# 启动前端开发服务器
npm run dev
# 浏览器访问 http://localhost:3000
```

---

## 🚀 服务器生产部署指南（Ubuntu）

生产环境架构：**Nginx 统一负责 80/443 入口与静态托管，Let's Encrypt 证书自动化守护，后端 systemd 仅监听 127.0.0.1**。

### 1. 后端 systemd 守护配置
创建系统服务文件：
```bash
sudo nano /etc/systemd/system/zero-backend.service
```
配置如下：
```ini
[Unit]
Description=Zero to Tech FastAPI Backend Service
After=network.target

[Service]
User=john
Group=john
WorkingDirectory=/home/john/zero-to-tech/backend

# 优化 1：强制 Python 关闭日志缓冲，让 journalctl 日志能够实时秒级打印出来
Environment="PYTHONUNBUFFERED=1"
Environment="PATH=/home/john/zero-to-tech/backend/.venv/bin:/usr/bin:/bin"

# 优化 2：
# - 绑定 127.0.0.1 阻断公网直接探测 8000 端口
# - 加上 --proxy-headers 准确透传真实客户端 IP
ExecStart=/home/john/zero-to-tech/backend/.venv/bin/uvicorn main:app --host 127.0.0.1 --port 8000 --proxy-headers

# 优化 3：崩溃自愈机制（异常退出 3 秒自动拉起）
Restart=always
RestartSec=3s

# 优化 4：基础安全与日志规范
PrivateTmp=true
SyslogIdentifier=zero-backend

[Install]
WantedBy=multi-user.target
```
启动并设为开机自启：
```bash
sudo systemctl daemon-reload
sudo systemctl start zero-backend
sudo systemctl enable zero-backend
```

### 2. 前端静态打包
```bash
cd ~/zero-to-tech

# 生产环境 API 前缀留空（走相对路径反代）
echo "NEXT_PUBLIC_API_BASE_URL=" > .env.production

npm install
npm run build   # 输出静态网页到 out/ 目录
```

### 3. Nginx 反向代理与安全兜底配置

#### (1) 配置默认 404 兜底防扫描（`/etc/nginx/sites-available/default`）
拦截任何未授权域名或裸 IP 直连：
```nginx
server {
    listen 80 default_server;
    listen [::]:80 default_server;
    server_name _;
    return 404;
}
```

#### (2) 配置主业务站点（`/etc/nginx/sites-available/zero-to-tech`）
将 `your-domain.com` 替换为你的真实域名（例如 `aaa.centralindia.cloudapp.azure.com`）：
```nginx
server {
    listen 80;
    server_name your-domain.com;

    # 1. 托管 Next.js 静态文件
    root /home/john/zero-to-tech/out;
    index index.html;

    # 页面路由匹配规则（先查静态 html，再兜底单页 index.html）
    location / {
        try_files $uri $uri.html $uri/ /index.html;
    }

    # 2. 将 /api/ 请求反向代理到 127.0.0.1:8000
    location /api/ {
        proxy_pass http://127.0.0.1:8000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

激活配置并检查：
```bash
sudo ln -sfn /etc/nginx/sites-available/default /etc/nginx/sites-enabled/default
sudo ln -sfn /etc/nginx/sites-available/zero-to-tech /etc/nginx/sites-enabled/zero-to-tech
sudo nginx -t
sudo systemctl reload nginx
```

### 4. 申请 HTTPS 免费证书（Certbot 自动化）
安装 Certbot 与 Nginx 插件：
```bash
sudo apt update
sudo apt install -y certbot python3-certbot-nginx
```

一键签发证书并自动配置全站 HTTPS 重定向：
```bash
sudo certbot --nginx -d your-domain.com
```
* Certbot 会自动修改 Nginx 增加 443 端口监听，并将 HTTP 80 请求以 301 状态自动跳转到 HTTPS。
* 系统自带的 `certbot.timer` 每日会自动检查证书，并在到期前 30 天无感静默续期。

---

## 🔄 日常维护与代码更新

### 代码发布流程
本地提交推送到 GitHub 后，登录服务器执行：

```bash
cd ~/zero-to-tech

# 1. 拉取最新代码
git pull origin main

# 2. 如果后端有改动，重启 systemd 服务
sudo systemctl restart zero-backend

# 3. 如果前端有改动，重新打包（Nginx 静态目录自动更新生效）
npm run build
```

### 常用运维排错命令
```bash
# 查看后端实时日志（毫秒级输出）
journalctl -u zero-backend -f

# 查看后端运行状态
sudo systemctl status zero-backend

# 测试 Nginx 配置文件语法
sudo nginx -t

# 检查 SSL 证书状态与有效期
sudo certbot certificates
```

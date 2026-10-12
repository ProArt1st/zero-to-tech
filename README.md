# Zero to Tech · 全栈中文文本实验室

本项目为「零到全栈」课程的作品集与文字实验室项目。从最初的静态 HTML/React 演进为 **Next.js 前端 + FastAPI 后端 + SQLite 存储** 的全栈架构，生产环境使用 **Nginx 反向代理 + systemd 服务守护**。

---

## 🛠️ 技术栈架构

* **前端**：Next.js 15 (`app/` 路由架构) + React 19 + 纯手写 CSS + Anime.js（静态导出 `out/`）
* **后端**：Python 3 + FastAPI + Uvicorn
* **算法与功能**：
  * **SnowNLP**：中文情绪打分分析
  * **PyPinyin**：汉字多音字注音与拼音转换
  * **Session**：基于 Cookie / UUID 的无状态会话跟踪
* **数据库**：SQLite 3 (`backend/history.db`) 持久化历史分析记录
* **服务器运维**：
  * **systemd**：以系统服务常驻守护 FastAPI（仅监听 `127.0.0.1:8000`）
  * **Nginx**：统一监听 80 端口，静态托管前端并反向代理 `/api/`，彻底消除跨域问题

---

## 📁 项目结构

```text
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
│   ├── main.py           # API 接口入口与 CORS 配置
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

生产环境推荐架构：**Nginx 对外统一暴露 80 端口，systemd 守护后端仅监听 127.0.0.1**。

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
WorkingDirectory=/home/john/zero-to-tech/backend
Environment="PYTHONUNBUFFERED=1"
Environment="PATH=/home/john/zero-to-tech/backend/.venv/bin:/usr/bin:/bin"
ExecStart=/home/john/zero-to-tech/backend/.venv/bin/uvicorn main:app --host 127.0.0.1 --port 8000 --proxy-headers
Restart=always
RestartSec=3s
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

### 3. Nginx 反向代理配置
编辑站点配置文件 `/etc/nginx/sites-available/zero-to-tech`：
```nginx
server {
    listen 80;
    server_name _;

    # 1. 托管 Next.js 静态文件
    root /home/john/zero-to-tech/out;
    index index.html;

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
激活配置并重启：
```bash
sudo ln -sf /etc/nginx/sites-available/zero-to-tech /etc/nginx/sites-enabled/
sudo rm -f /etc/nginx/sites-enabled/default
sudo nginx -t
sudo systemctl reload nginx
```

---

## 🔄 日常代码更新流程

本地提交推送到 GitHub 后，在服务器上只需三步：

```bash
cd ~/zero-to-tech

# 1. 拉取最新代码
git pull origin main

# 2. 如果后端有改动，重启 systemd
sudo systemctl restart zero-backend

# 3. 如果前端有改动，重新打包（Nginx 自动生效，无需重启 Nginx）
npm run build
```

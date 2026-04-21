# Binance Square 热点抓取 Skills

基于 [Hermes Agent](https://github.com/NousResearch/hermes-agent) + [browser-harness](https://github.com/browser-use/browser-use) 的币安广场（Binance Square）热点话题自动抓取工具集。

自动抓取当日热点话题、高讨论度帖子、热搜币种、恐惧贪婪指数，生成精美暗色主题 HTML 报告，所有条目均可点击直达原帖。

---

## 📸 效果预览

- 暗色主题（背景 `#0b0e11`，符合币安风格）
- 热门话题卡片（排名徽章 + 浏览量 + 讨论人数）
- 帖子卡片（作者 + 时间 + 正文摘要 + 「查看原帖 →」按钮）
- 热搜币种（涨跌幅 + 链接直达交易页）
- 恐惧贪婪指数
- 悬停金色边框 + 位移动画

---

## 🛠 必装依赖

### 1. Hermes Agent
```bash
pip install hermes-agent
# 或克隆源码
git clone https://github.com/NousResearch/hermes-agent
cd hermes-agent && pip install -e .
```

### 2. browser-harness
```bash
pip install browser-use
# 安装 CLI（确保 browser-harness 在 $PATH）
```

> 详见 browser-harness 官方文档：https://github.com/browser-use/browser-use

### 3. Google Chrome
必须安装 Chrome（用于 CDP 远程调试），推荐 v120+。

macOS 安装：
```bash
brew install --cask google-chrome
```

### 4. python-socks（macOS 代理用户必装）
如果你使用了系统 SOCKS 代理（如 Clash、V2Ray 等），必须安装：
```bash
pip install python-socks
```

---

## 🚀 快速开始

### 第一步：启动带调试端口的 Chrome

```bash
# macOS
nohup "/Applications/Google Chrome.app/Contents/MacOS/Google Chrome" \
  --headless=new \
  --remote-debugging-port=9222 \
  --user-data-dir=/tmp/chromedebug \
  --no-sandbox \
  --disable-gpu \
  --disable-dev-shm-usage > /tmp/chrome.log 2>&1 &

sleep 5

# 获取 WebSocket 调试 URL
WS_URL=$(curl -s http://localhost:9222/json/version | python3 -c "import sys,json; print(json.load(sys.stdin)['webSocketDebuggerUrl'])")
export BU_CDP_WS="$WS_URL"
echo "Chrome 已就绪：$WS_URL"
```

### 第二步：安装 Skills 到 Hermes

将本仓库中的 skill 文件复制到 Hermes skills 目录：

```bash
# 复制 social-media skill
mkdir -p ~/.hermes/skills/social-media/binance-square
cp skills/social-media/binance-square-skill.md ~/.hermes/skills/social-media/binance-square/SKILL.md

# 复制 browser-harness scraper skill
mkdir -p ~/.hermes/skills/browser-harness/binance-square-scraper
cp skills/browser-harness/binance-square-scraper-skill.md ~/.hermes/skills/browser-harness/binance-square-scraper/SKILL.md

# 复制 browser-harness 主 skill（如果没有）
cp skills/browser-harness/browser-harness-skill.md ~/.hermes/skills/browser-harness/SKILL.md

# 复制 domain skill
mkdir -p ~/.hermes/skills/browser-harness/domain-skills/binance-square
cp skills/browser-harness/domain-skills/binance-square/scraping.md ~/.hermes/skills/browser-harness/domain-skills/binance-square/scraping.md
```

### 第三步：触发 Hermes

在 Hermes 对话框中输入：

```
币安广场今日热点
```

或：

```
帮我抓取币安广场热点生成报告
```

Hermes 会自动加载 skill → 导航到币安广场 → 抓取数据 → 生成 HTML 报告保存到 `~/Documents/Hermes/binance_square_今日热点.html`。

---

## 📁 文件结构

```
binance-square/
├── README.md
└── skills/
    ├── social-media/
    │   └── binance-square-skill.md       # 主触发 skill（触发词：币安广场）
    └── browser-harness/
        ├── browser-harness-skill.md      # browser-harness 基础 skill
        ├── binance-square-scraper-skill.md  # 详细抓取步骤 + HTML 报告规范
        └── domain-skills/
            └── binance-square/
                └── scraping.md           # 稳定选择器 + 已知坑点
```

---

## ⚙️ 配置说明

### 输出路径

默认 HTML 报告保存到 `~/Documents/Hermes/`，可在 `binance-square-scraper-skill.md` 中修改：

```markdown
## 输出要求（用户偏好）
输出 HTML 报告，保存到 `~/Documents/Hermes/`
```

改为你想要的路径即可。

### Chrome 路径（Linux/Windows）

Linux：
```bash
nohup google-chrome --headless=new --remote-debugging-port=9222 --user-data-dir=/tmp/chromedebug &
```

Windows（PowerShell）：
```powershell
Start-Process "C:\Program Files\Google\Chrome\Application\chrome.exe" `
  -ArgumentList "--headless=new --remote-debugging-port=9222 --user-data-dir=C:\tmp\chromedebug"
```

---

## 🔑 关键技术说明

### 为什么用 CDP 而不是 Playwright/Selenium？

币安广场是 React SPA，`js()` helper 直接返回 `None`，必须使用 `cdp("Runtime.evaluate", ...)` 提取 DOM 数据。

### 为什么用 JS 导航而不是 `goto()`？

`goto()` 在币安广场会**无限挂起**（页面永不触发 load 事件），必须用：
```python
js("window.location.href = 'https://www.binance.com/zh-CN/square'")
time.sleep(8)
```

### Chrome UUID 变化问题

Chrome 每次重启后 WebSocket URL 中的 browser UUID 会变，必须重新获取并设置 `BU_CDP_WS` 环境变量。上面的启动脚本已处理此问题。

---

## 🐛 常见问题

| 问题 | 原因 | 解决方案 |
|------|------|----------|
| `connecting through a SOCKS proxy requires python-socks` | 系统 SOCKS 代理 | `pip install python-socks` |
| `requires non-default data directory` | Chrome 用了真实 user-data-dir | 改用 `--user-data-dir=/tmp/chromedebug` |
| `wait_for_load()` 卡死 | 币安广场不触发 load 事件 | 改用 `time.sleep(8)` |
| 数据为空 | 页面还没渲染完 | 增加等待时间至 10s |
| 话题链接乱码 | 中文被 URL encode | 直接从链接列表取 href，不要手动拼接 |

---

## 📝 License

MIT

---

## 🙏 致谢

- [Hermes Agent](https://github.com/NousResearch/hermes-agent) — Nous Research 出品的 AI Agent 框架
- [browser-use](https://github.com/browser-use/browser-use) — browser-harness 底层驱动
- [Binance Square](https://www.binance.com/zh-CN/square) — 数据来源

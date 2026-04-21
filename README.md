# 币安广场热点抓取 Skills

> 基于 [Hermes Agent](https://github.com/NousResearch/hermes-agent) + [browser-harness](https://github.com/browser-use/browser-harness) 实现自动抓取币安广场（Binance Square）今日热点话题、高讨论帖子、热搜币种，并生成精美 HTML 热点报告。

![demo](https://img.shields.io/badge/platform-Binance%20Square-f0b90b?style=flat-square&logo=binance)
![lang](https://img.shields.io/badge/language-Python-3776AB?style=flat-square&logo=python)
![license](https://img.shields.io/badge/license-MIT-green?style=flat-square)

---

## 📸 效果预览

生成的 HTML 报告包含：
- 🔥 热门话题 TOP N（含浏览量、讨论人数，整卡可点击跳转话题页）
- 💬 高讨论度帖子（含作者、摘要、互动数，整卡可点击跳转原帖）
- 📈 热搜币种（含价格、涨跌幅，可点击跳转币安交易页）
- 😨 恐惧贪婪指数
- 暗色主题 + 金色悬停动效（符合币安 #f0b90b 风格）

---

## 🧩 Skills 结构

```
skills/
├── social-media/
│   └── binance-square-trigger.md        # 触发词 & 执行入口 Skill
├── browser-harness/
│   ├── browser-harness-core.md          # browser-harness 核心 Skill（连接/使用规范）
│   ├── binance-square-scraper/
│   │   └── SKILL.md                     # 币安广场抓取专项 Skill（CDP 提取）
│   └── domain-skills/binance-square/
│       └── scraping.md                  # 字段测试得出的稳定选择器 & 坑点记录
```

---

## 🛠️ 必装依赖

### 1. Hermes Agent

Hermes Agent 是承载 Skills 的 AI 助手框架，所有 Skill 在其上运行。

```bash
# 安装 Hermes Agent（需要 Python 3.11+）
pip install hermes-agent
# 或从源码安装
git clone https://github.com/NousResearch/hermes-agent
cd hermes-agent && pip install -e .
```

### 2. browser-harness

browser-harness 是通过 CDP（Chrome DevTools Protocol）直接控制浏览器的工具。

```bash
# 需要 uv（推荐）
brew install uv    # macOS
# 或
pip install uv

# 安装 browser-harness
git clone https://github.com/browser-use/browser-harness ~/Developer/browser-harness
cd ~/Developer/browser-harness
uv tool install -e .
```

> ⚠️ browser-harness 以 **可编辑模式** 安装，`~/Developer/browser-harness` 目录必须保留，不能删除。

### 3. Google Chrome

browser-harness 需要连接到已运行的 Chrome，并开启远程调试端口。

```bash
# macOS 启动带调试端口的 Chrome（关键！）
/Applications/Google\ Chrome.app/Contents/MacOS/Google\ Chrome \
  --remote-debugging-port=9222 \
  --user-data-dir=/tmp/chrome-debug \
  --remote-allow-origins=* &
```

> 如果 Chrome 已经运行，需先退出再用上面命令启动。

### 4. python-socks（可选，macOS 代理用户必装）

如果你的 macOS 开启了系统 SOCKS 代理（如 127.0.0.1:7890），需安装此包，否则 browser-harness 连接 localhost 会报错：

```bash
uv pip install --python ~/.local/share/uv/tools/harness/bin/python python-socks
```

---

## 🚀 快速上手

### Step 1：克隆本仓库，将 Skills 安装到 Hermes

```bash
git clone https://github.com/itgoyo/binance-square
cd binance-square

# 将 skills 复制到 Hermes skills 目录
cp -r skills/social-media/binance-square-trigger.md \
  ~/.hermes/skills/social-media/binance-square/SKILL.md

cp -r skills/browser-harness/binance-square-scraper/SKILL.md \
  ~/.hermes/skills/browser-harness/binance-square-scraper/SKILL.md

cp skills/browser-harness/browser-harness-core.md \
  ~/.hermes/skills/browser-harness/SKILL.md

mkdir -p ~/.hermes/skills/browser-harness/domain-skills/binance-square
cp skills/browser-harness/domain-skills/binance-square/scraping.md \
  ~/.hermes/skills/browser-harness/domain-skills/binance-square/scraping.md
```

### Step 2：启动 Chrome（调试模式）

```bash
/Applications/Google\ Chrome.app/Contents/MacOS/Google\ Chrome \
  --remote-debugging-port=9222 \
  --user-data-dir=/tmp/chrome-debug \
  --remote-allow-origins=* &
sleep 3
# 验证连接
curl -s http://localhost:9222/json/version | grep webSocketDebuggerUrl
```

### Step 3：在 Hermes 中触发

启动 Hermes Agent 后，直接输入以下任一触发词：

```
币安广场今日热点
帮我抓取币安广场热点生成报告
币安广场
```

Hermes 会自动：
1. 连接 Chrome
2. 导航到 `https://www.binance.com/zh-CN/square`
3. 抓取话题 / 帖子 / 热搜币种
4. 生成 HTML 报告并自动用浏览器打开

**报告默认保存到** `~/Documents/Hermes/binance_square_今日热点.html`

---

## 🔧 工作原理

### 为什么用 browser-harness？

币安广场是重度 React SPA：
- `goto()` / `wait_for_load()` 会**无限挂起**
- 必须用 `JS 导航` + `time.sleep(7-8)` 等待渲染
- DOM 数据提取需通过 `cdp("Runtime.evaluate", ...)` 走 CDP 协议

### 核心抓取逻辑

```python
# 1. JS 导航（不用 goto/new_tab）
js("window.location.href = 'https://www.binance.com/zh-CN/square'")
time.sleep(8)

# 2. 提取侧边栏（话题 + 热搜币种 + 恐惧贪婪指数）
sidebar = cdp("Runtime.evaluate",
    expression="document.querySelector('.feed-layout-widget-sidebar').innerText",
    returnByValue=True)

# 3. 提取帖子（用稳定的 URL 选择器）
posts = cdp("Runtime.evaluate", expression="""
  Array.from(document.querySelectorAll('a[href*="/square/post/"]'))
    .map(a => ({href: a.href, text: a.closest('article')?.innerText || ''}))
    .slice(0, 15)
""", returnByValue=True)

# 4. 滚动加载更多
cdp("Runtime.evaluate", expression="window.scrollBy(0, 2000)", returnByValue=True)
time.sleep(3)
```

### 稳定 URL 格式

| 类型 | URL 格式 |
|------|---------|
| 帖子原文 | `https://www.binance.com/zh-CN/square/post/{postId}` |
| 话题页 | `https://www.binance.com/zh-CN/square/hashtag/{topic}` |
| 用户主页 | `https://www.binance.com/zh-CN/square/profile/{handle}` |
| 现货交易 | `https://www.binance.com/zh-CN/trade/{COIN}_USDT` |
| 合约交易 | `https://www.binance.com/zh-CN/futures/{COIN}USDT` |

---

## ⚠️ 常见问题

### Q: `js()` 返回 None？
A: 币安广场的 `js()` helper 无效，必须用 `cdp("Runtime.evaluate", expression="...", returnByValue=True)` 路径。

### Q: 页面加载后抓到空内容？
A: 等待时间不够。币安广场 SPA 首屏需要 6~8 秒渲染，`time.sleep(8)` 是最低要求。

### Q: `browser-harness` 连接报 `python-socks` 错误？
A: macOS 系统代理（如 ClashX/Surge）拦截了 localhost 连接。运行：
```bash
uv pip install --python ~/.local/share/uv/tools/harness/bin/python python-socks
```

### Q: Chrome 启动后还是连不上？
A: Chrome 重启后 WS URL 会变，需要重新获取：
```bash
WS_URL=$(curl -s http://localhost:9222/json/version | python3 -c "import sys,json; print(json.load(sys.stdin)['webSocketDebuggerUrl'])")
export BU_CDP_WS="$WS_URL"
```

### Q: CSS class 选择器失效？
A: 币安使用 CSS Modules（class 名含 hash，每次构建都会变）。本项目统一用 URL 选择器（`a[href*="/square/post/"]`）和 `.innerText` 文本提取，不依赖 class 名，稳定可靠。

---

## 📄 License

MIT License — 自由使用、修改、分发。

---

## 🙏 致谢

- [Hermes Agent](https://github.com/NousResearch/hermes-agent) — Nous Research 出品的 AI 助手框架
- [browser-harness](https://github.com/browser-use/browser-harness) — CDP 浏览器直控工具
- [Binance Square](https://www.binance.com/zh-CN/square) — 数据来源

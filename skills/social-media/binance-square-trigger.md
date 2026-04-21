---
name: binance-square
description: 抓取币安广场（Binance Square）今日热点话题和高讨论度帖子，生成带可点击跳转链接的 HTML 热点报告，保存到 ~/Documents/Hermes/。
tags: [binance, crypto, scraping, browser-harness]
related_skills: [browser-harness]
---

# 币安广场热点报告

## 触发条件

用户输入「**币安广场**」时立即执行本 Skill，无需额外确认。

## 目标

抓取 `https://www.binance.com/zh-CN/square` 当天热点话题 + 讨论度最高的帖子，输出一份精美 HTML 报告，所有条目均可点击直达原帖/话题页。

---

## 执行步骤

### 1. 加载 browser-harness skill

先用 `skill_view(name="browser-harness")` 读取完整 skill 确认连接方式，再执行后续步骤。

### 2. 用 JS 导航打开币安广场（避免 goto/new_tab hang）

```python
import time, json
js("window.location.href = 'https://www.binance.com/zh-CN/square'")
time.sleep(8)
url = js("window.location.href")
print("landed:", url)
screenshot()
```

### 3. 截图确认页面结构，再滚动抓取

```python
screenshot()           # 确认已加载
# ⚠️ scroll_down() 在 browser-harness 中不存在！用 JS 滚动：
js("window.scrollBy(0, 800)")
time.sleep(1)
js("window.scrollBy(0, 800)")
time.sleep(1)
js("window.scrollBy(0, 800)")
time.sleep(2)
screenshot()
```

### 4. 用 JS 提取热点话题（Topics / Trending）

```python
# 热点话题区块：包含话题名、浏览量、讨论人数、话题链接
topics_raw = js("""
  const items = [];
  // 尝试各种选择器（广场 SPA 经常改 class）
  const cards = document.querySelectorAll('[class*="topic"], [class*="trending"], [class*="hot"]');
  cards.forEach(c => {
    const a = c.querySelector('a[href]');
    const text = c.innerText;
    if (a && text) items.push({href: a.href, text: text.trim()});
  });
  JSON.stringify(items.slice(0, 10));
""")
print(topics_raw)
```

若上述选择器为空，改用截图 → `click(x, y)` 定位 + `page_info()` 读取可见文本。

### 5. 提取帖子列表（Post Cards）

```python
posts_raw = js("""
  const posts = [];
  // Binance Square 帖子卡片通常带 data-track 或 article 标签
  const cards = document.querySelectorAll('article, [class*="post-card"], [class*="feedCard"], [class*="feed-item"]');
  cards.forEach(c => {
    const a = c.querySelector('a[href*="/square/post/"]');
    const author = c.querySelector('[class*="author"], [class*="name"], [class*="username"]');
    const content = c.querySelector('[class*="content"], [class*="body"], p');
    const views = c.querySelector('[class*="view"], [class*="read"]');
    const likes = c.querySelector('[class*="like"], [class*="react"]');
    if (a) posts.push({
      href: a.href,
      author: author ? author.innerText.trim() : '',
      content: content ? content.innerText.trim().slice(0, 150) : c.innerText.trim().slice(0, 150),
      views: views ? views.innerText.trim() : '',
      likes: likes ? likes.innerText.trim() : ''
    });
  });
  JSON.stringify(posts.slice(0, 15));
""")
print(posts_raw)
```

### 6. 兜底方案：用 page_info() 兜底

若 JS 提取结果为空或 `[]`，则：
```python
info = page_info()
print(info)  # 从纯文本中手动解析帖子标题、作者、链接
```

### 7. 生成 HTML 报告

将抓取到的数据组装成以下 HTML 模板，写入文件：

```
~/Documents/Hermes/binance_square_今日热点.html
```

---

## HTML 模板规范

- **暗色主题**：背景 `#0b0e11`，卡片 `#1a1d21`，强调色金黄 `#f0b90b`
- **话题卡片**：展示话题名、浏览量、讨论人数，`<a href="话题链接" target="_blank">` 整卡可点击
- **帖子卡片**：展示作者名、摘要（≤150字）、浏览量/点赞数，`<a href="/square/post/xxx" target="_blank">` 整卡可点击
- **鼠标悬停效果**：`border: 1px solid #f0b90b; transform: translateY(-2px);` 
- **页眉**：币安广场 Logo 文字 + 今日日期
- **响应式**：卡片 grid，`grid-template-columns: repeat(auto-fill, minmax(300px, 1fr))`
- **所有外部链接**加 `target="_blank" rel="noopener"`

---

## 关键注意事项（Pitfalls）

1. **导航用 JS 不用 goto/new_tab**：Binance Square 是重度 SPA，`goto()` 和 `new_tab()` + `wait_for_load()` 均可能无限挂起。固定用：
   ```python
   js("window.location.href = 'https://www.binance.com/zh-CN/square'")
   time.sleep(7)
   ```

2. **等待要充分**：广场首屏需 6~8 秒才渲染完帖子列表，不够会抓到空内容。

3. **Class 名经常变**：Binance 前端用 CSS Modules（class 含 hash），优先用 `[class*="xxx"]` 模糊匹配，或用语义标签 `article`、`a[href*="/square/post/"]`。

4. **帖子链接格式**：`https://www.binance.com/zh-CN/square/post/{postId}`，注意语言前缀。

5. **话题链接格式**：`https://www.binance.com/zh-CN/square/topic/{topicId}`。

6. **浏览器连接**：先检查 `/tmp/bu-default.sock` 是否存在；若 daemon 挂了用 `restart_daemon()` 重启。

7. **`scroll_down()` 不存在**：browser-harness helpers 里没有 `scroll_down` 函数，调用会报 `NameError`。必须用 JS：`js("window.scrollBy(0, 800)")`。

8. **`page_info()` 不支持切片**：`page_info()` 返回的不是字符串，不能用 `page_info()[:5000]`，会报 `KeyError`。要截断请先赋值再切片：`info = page_info(); print(str(info)[:5000])`。

9. **帖子链接用 `a[href*="/square/post/"]` 选择器**：比 `article` 等语义标签更可靠，Binance Square SPA 用 CSS Modules（class 含 hash），但帖子链接 URL 格式稳定。

10. **话题链接用 `a[href*="hashtag"]` 或 `a[href*="topic"]`** 比 `[class*="topic"]` 更稳定，class 名经常变但 URL 结构不变。

7. **输出文件路径**：固定为 `~/Documents/Hermes/binance_square_今日热点.html`，用 `write_file()` 写入。

8. **生成完毕后** 用 `terminal("open ~/Documents/Hermes/binance_square_今日热点.html")` 自动在浏览器打开预览。

---

## 最终输出格式

生成 HTML 后，在对话中用 Markdown 表格汇总：

```
## 📊 今日币安广场热点汇总（YYYY-MM-DD）

### 🔥 热门话题 TOP N
| 排名 | 话题 | 浏览量 | 讨论人数 |
...

### 💬 高讨论度帖子精选（均可点击直达原帖）
1. **作者** — 内容摘要（时间/浏览量）
...

文件已保存：`~/Documents/Hermes/binance_square_今日热点.html`
```

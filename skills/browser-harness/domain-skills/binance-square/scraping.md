# Binance Square — Scraping Guide

URL: `https://www.binance.com/zh-CN/square`

## Navigation

`goto()` **hangs** on Binance Square. Use JS navigation:
```python
js("window.location.href = 'https://www.binance.com/zh-CN/square'")
time.sleep(7)
```

## Page Layout

```
.feed-layout-main          — main feed (posts)
.feed-layout-nav-sidebar   — left nav (发现/新闻/通知/书签…)
.feed-layout-widget-sidebar — right sidebar (topics, coins, sentiment)
```

## Stable Selectors

### Posts (feed items)
Posts do NOT have a single stable wrapper class. Use post link anchors as pivot:
```javascript
// All post links
document.querySelectorAll('a[href*="/square/post/"]')

// Per-post: walk up from anchor to container, grab innerText
let links = Array.from(document.querySelectorAll('a[href*="/square/post/"]'));
let seen = new Set();
links.forEach(a => {
  if (seen.has(a.href)) return;
  seen.add(a.href);
  let container = a.closest('[class*="feed"], article') || a.parentElement.parentElement.parentElement;
  // container.innerText has full post text + stats
});
```

### Hot Topics (right sidebar)
```javascript
// Topic hashtag links — href contains /square/hashtag/
document.querySelectorAll('a[href*="hashtag"], a[href*="topic"]')
// Each link's closest div has: topic name, view count, participant count
```

### Hot Coins (right sidebar)
```javascript
// Trade links in sidebar
document.querySelectorAll('.feed-layout-widget-sidebar a[href*="trade"]')
// Sibling text has: coin name, price, % change, heat label
```

### Full sidebar text (fastest for sentiment + coins)
```javascript
document.querySelector('.feed-layout-widget-sidebar').innerText
// Contains: fear/greed index, top searched coins, "you may know" creators
```

### Full feed text (fastest for post content)
```javascript
document.querySelector('.feed-layout-main').innerText.slice(0, 5000)
```

## Known Gotchas

- **Posts have no stable wrapper class** — CSS modules generate hashed names. Pivot on `a[href*="/square/post/"]` instead.
- **Coin price/change classes are hashed** — read them from sidebar `.innerText` by regex rather than by selector.
- **Page is a React SPA** — wait at least 7s after JS navigation before querying DOM.
- **Hot topics expand on hover** — static scrape gets full description text inline in sidebar HTML.
- **`screenshot()` + vision** is a useful fallback when selectors return empty (e.g. lazy-loaded sections not yet in viewport).

## Sample Output Shape

```json
{
  "posts": [
    {"content": "...", "link": "https://www.binance.com/zh-CN/square/post/314390396983922"}
  ],
  "topics": [
    {"text": "#美伊冲突...\n52663次浏览\n328人讨论", "href": "https://www.binance.com/zh-CN/square/hashtag/..."}
  ],
  "coins": [
    {"symbol": "HIGH", "price": "0.38", "change": "+50.81%", "trade_url": "https://www.binance.com/zh-CN/trade/HIGH_USDT"}
  ]
}
```

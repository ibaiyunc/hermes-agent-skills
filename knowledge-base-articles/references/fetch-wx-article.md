# fetch-wx-article 参考

> 工具：ClawHub skill `fetch-wx-article/scripts/fetch_wx_article.py`
> 用途：通过 scrapling 直接抓取微信公开文章正文，无需登录

## 路径

```
~/.openclaw/workspace/skills/fetch-wx-article/scripts/fetch_wx_article.py
```

## 使用方式

```bash
/usr/bin/python3 ~/.openclaw/workspace/skills/fetch-wx-article/scripts/fetch_wx_article.py \
  "https://mp.weixin.qq.com/s/GsCaK9YeWBU16-wFvBFmRw"
```

**必须用系统 Python `/usr/bin/python3`**，不是 hermes venv 的 python3。

## 依赖

```bash
/usr/bin/pip3 install scrapling html2text curl_cffi browserforge
```

- `scrapling` 是主力解析库，不需要浏览器环境
- 依赖缺失时逐步补装即可

## 成功率与已知失败模式

| 情况 | 处理方式 |
|------|---------|
| 正常文章 | 直接返回完整纯文本（5000~8000 字节量级） |
| 返回 56 字节（只有 UI 噪声） | **单独重试一次**，通常成功 |
| 多图文章 | 图片链接无法获取，只保留文字 |
| 小说阅读器嵌入 | 正文完整，标题可能有 DOM 污染，用 `<title>` 兜底 |

## 并发抓取

直接用 Python `ThreadPoolExecutor` 并发调用该脚本，`max_workers=4` 足够：

```python
from concurrent.futures import ThreadPoolExecutor, as_completed
import subprocess

script = os.path.expanduser("~/.openclaw/workspace/skills/fetch-wx-article/scripts/fetch_wx_article.py")

def fetch_one(idx_url):
    idx, url = idx_url
    r = subprocess.run(["/usr/bin/python3", script, url],
                       capture_output=True, text=True, timeout=60)
    return idx, url, r.stdout  # r.stdout 就是解析好的纯文本

with ThreadPoolExecutor(max_workers=4) as executor:
    futures = {executor.submit(fetch_one, u): u for u in urls}
    for future in as_completed(futures):
        idx, url, text = future.result()
        # text 已解析好，直接 parse_article(text) 即可
```

## 输出格式

脚本输出的是**解析好的纯文本**，格式如下：

```
# 文章标题

Original 账号名 账号名 账号名

[正文...]
```

- 标题在第一行（`# ` 前缀）
- `Original` 行是账号名重复三遍，之后是正文
- 正文包含 Markdown 格式（`**粗体**`、`## 标题`、代码块等）

---

## ⚠️ 浏览器自动化方案无法绕过微信

**结论**：所有浏览器自动化方案（CloakBrowser、Playwright + stealth args、humanize 鼠标拟人化）对微信的"环境异常"页面均无效。

**实测结果（2026-05-28，公众号文章 `mp.weixin.qq.com`）：

| 方案 | 结果 |
|------|------|
| fetch-wx-article.py（scrapling） | ✅ 成功 |
| 原版 Playwright | ❌ "环境异常" 拦截页 |
| Playwright + CloakBrowser stealth args | ❌ "环境异常" 拦截页 |
| CloakBrowser `launch_async`（复用 Playwright Chromium） | ❌ "环境异常" 拦截页 |

CloakBrowser 文章（19000+ Star）声称能过 Google reCAPTCHA 0.9 分和 Cloudflare 人机验证，但微信自研的浏览器指纹 + 环境检测比这两家更严，无法穿透。

**正确工作流永远是**：`fetch-wx-article.py`（scrapling 引擎） > 任何浏览器方案。

---

## ⚠️ 系统代理导致 `ERR_PROXY_CONNECTION_FAILED`

如果浏览器自动化测试时遇到 `net::ERR_PROXY_CONNECTION_FAILED` 且没有运行任何代理服务，先检查系统级代理：

```bash
# 查看当前代理设置
gsettings get org.gnome.system.proxy mode        # 应为 'none'
gsettings get org.gnome.system.proxy.http host  # 确认无残留

# 关闭系统代理（不需要杀 Mihomo）
gsettings set org.gnome.system.proxy mode 'none'
```

常见坑：Mihomo/Clash 可能已退出，但 GNOME 系统代理设置仍指向 `127.0.0.1:7890`，导致浏览器流量被导向已经不存在的本地端口而报 `ERR_PROXY_CONNECTION_FAILED`。关掉 gsettings 即可。

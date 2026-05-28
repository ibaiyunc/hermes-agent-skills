---
name: knowledge-base-articles
description: 知识库文章采集与保存工作流——将微信公众号、博客、论文等外部文章规约化地存入本地知识库，包含分类判断、存放路径、文件命名、去重检查全流程。
version: 1.0.0
author: Hermes Agent
metadata:
  tags: [knowledge-base, article-save, web-scraping, notes]
  trigger: 用户要求将某个 URL 文章保存到知识库时激活
---

# 知识库文章采集与保存

## 触发条件

用户说"放到知识库"、"保存到知识库"、"收藏这篇文章"等类似指令时激活。

## 存放路径规范

**正确路径**：`/home/baiyun/obsidian-vault/10-Notes/<分类>/`

**Obsidian Vault 结构**：
```
/home/baiyun/obsidian-vault/
├── 00-Inbox/Hermes/     # Hermes 写入的临时文件
├── 10-Notes/            # 知识库（主要存放位置）
│   ├── knowledge-index.md   # 全局索引文件（必须同步更新）
│   ├── arm-reverse/
│   ├── app-reverse/
│   └── <新建分类>/
└── 90-Archive/          # 归档
```

**禁止路径**：`/home/baiyun/novels/` —— 小说创作专用目录，禁止混入知识类文章。

### 现有分类目录

| 分类目录 | 内容类型 |
|---------|---------|
| `arm-reverse` | ARM / Android Native 逆向 |
| `app-reverse` | APP 抓包、SSL Pinning、Android 逆向 |
| `network-proxy` | 网络代理、Cloudflare Worker、爬虫技术、反检测浏览器 |
| `ai-agent` | AI Agent、MCP、免杀、工具推荐 |
| `ai-reverse` | AI 逆向、AST 反混淆、代码分析 |
| `windows-reverse` | Windows 逆向、ImHex、IDA Pro |
| `ctf-everyday` | CTF 每日挑战 |
| `quantum-computing` | 量子计算 |

如文章不属于现有分类，可在 `/home/baiyun/knowledge/` 下新建分类目录。

### 文件命名规范

```
00-<文章标题/主题>.md       ← 第一个文件
01-<文章标题/主题>.md       ← 第二个文件
02-…
```

## 工作流

1. **判断分类**：根据文章主题选择或新建分类目录
2. **抓取内容**：按下方"微信文章抓取"优先顺序尝试，抓到纯文本后直接进入解析步骤
3. **解析纯文本**：fetch-wx-article.py 或 CloakBrowser 输出格式均为 `# 标题\n\nOriginal 账号名\n\n正文...`，直接按行解析：
   ```python
   lines = text.strip().split('\n')
   title = next((l.strip().lstrip('# ').strip() for l in lines
                  if l.strip().lstrip('# ').strip()
                  and not l.strip().startswith('Original')
                  and len(l.strip()) > 3), "")
   author_m = re.search(r'^Original\s+(\S+)', text, re.MULTILINE)
   author = author_m.group(1) if author_m else ""
   ```
4. **结构化保存**：转写为 Markdown 格式，包含文件头元数据 + 正文
5. **同分类下去重**：检查目录中是否已有同名或同主题文件，如有则改用 `01-`、`02-` 前缀

## 微信文章抓取（优先顺序）

微信公号文章有验证码墙（直接 curl 或 playwright 访问 `mp.weixin.qq.com` 大概率返回"环境异常"）。

**优先顺序**：
1. **CloakBrowser `launch_async` + Mihomo 代理**（唯一能过微信"环境异常"的浏览器方案）
2. `fetch-wx-article.py`（scrapling 引擎，不需要浏览器，不需要登录）
3. `weixin-reader-oc`（备选）
4. OpenClaw 搜索 zone.ci 等镜像站
5. 均失败 → 请用户手动复制全文

### 方案1：CloakBrowser（首选，推荐）

需要 Mihomo 代理可用。二进制下载后缓存，后续无需代理。

**Step 1：启动 Mihomo 代理**
```bash
/opt/clash-party/resources/sidecar/mihomo \
  -d /home/baiyun/.config/mihomo-party/work \
  -f /home/baiyun/.config/mihomo-party/work/config.yaml &

# 设置系统代理
gsettings set org.gnome.system.proxy mode 'manual'
gsettings set org.gnome.system.proxy.http host '127.0.0.1'
gsettings set org.gnome.system.proxy.http port 7890
gsettings set org.gnome.system.proxy.https host '127.0.0.1'
gsettings set org.gnome.system.proxy.https port 7890
```

**Step 2：安装 CloakBrowser**
```bash
/usr/bin/pip3 install cloakbrowser
```

**Step 3：用 `launch_async` 抓取**（必须用异步版本，不是 `launch()`）
```python
import asyncio
from cloakbrowser import launch_async

async def fetch_wechat(url):
    browser = await launch_async(
        headless=True,
        humanize=True,     # 鼠标/键盘/滚动拟人化
        stealth_args=True   # 58 处 C++ 层指纹修改
    )
    page = await browser.new_page()
    resp = await page.goto(url, wait_until="domcontentloaded", timeout=20000)
    await page.wait_for_timeout(3000)
    text = await page.inner_text("body")
    await browser.close()
    if "环境异常" in text:
        raise RuntimeError("被微信拦截")
    return text

result = asyncio.run(fetch_wechat("https://mp.weixin.qq.com/s/xxxxx"))
```

详细工作流见 `references/wechat-browser-access.md`。

### 方案2：fetch-wx-article.py（备选）

```bash
/usr/bin/python3 ~/.openclaw/workspace/skills/fetch-wx-article/scripts/fetch_wx_article.py "<URL>"
```

输出格式：`# 标题\n\nOriginal 账号名\n\n正文...`，按行解析即可。

> **注意**：CloakBrowser 安装后自带快速验证脚本：
> ```python
> # 验证是否可用（测试微信文章能否打开）
> import asyncio; from cloakbrowser import launch_async
> asyncio.run((lambda: launch_async(headless=True, humanize=True))())
> ```

### 已知失败模式与处理

| 情况 | 处理方式 |
|------|---------|
| CloakBrowser 下载超时 | 确认 Mihomo 代理已启动且端口 7890 可用 |
| CloakBrowser 复用 Playwright Chromium 报错 | 必须用 `launch_async()`，不是 `launch()` |
| fetch-wx-article 返回极短内容（56 字节） | **单独重试一次**，成功率较高 |
| 多图文章 | 图片链接无法获取，只保留文字 |
| 标题被 DOM 污染（出现"在小说阅读器读本章"） | 取 `text.split('\n')[0].lstrip('# ')` 兜底 |

**注意**：部分文章图片无法直接获取，只保留文字内容。

## Pitfalls

## 入库后必做：同步索引

新增文章后，**必须**同步更新 `/home/baiyun/obsidian-vault/10-Notes/knowledge-index.md`，在对应分类的"关键词索引"表格中追加一行，包含：
- 文章标题/关键词
- 文件路径
- 核心内容标签

步骤：
1. 在分类章节添加新的文件条目（格式：`- \`文件名.md\` — 简介`）
2. 在关键词索引表格追加一行（格式：`|| 关键词 | 路径 |`）
3. 如新建了分类目录，在分类章节头部添加新分类的说明块

## 入库后维护（标签、检索、巡检）

详见 `references/vault-maintenance.md`：
- 语义搜索 + 混合检索 + 标签推荐 + 智能摘要
- 去重检测 + 损坏链接 + 标签一致性 + 知识谱系图
- 所有脚本路径 + cronjob 配置
- **关键陷阱**：Ollama nomic 欧氏距离换算、ChromaDB delete API、Graphviz render() 参数

## Pitfalls

- **勿混用目录**：`novels/` ≠ `obsidian-vault/10-Notes/`。存错一次就要移动文件并重新写入
- **同名覆盖**：同一目录下两个文件不能同名，先检查 `ls /home/baiyun/obsidian-vault/10-Notes/<分类>/`
- **hermes venv Python 无法安装某些包**：`/home/baiyun/.hermes/hermes-agent/venv/bin/python3` 是隔离环境，用 `pip3 install` 安装的包不在系统路径里。遇到 `ModuleNotFoundError` 时，先 `which python3` 确认——如果指向 hermes venv，改用 `/usr/bin/python3`，pip 也相应改为 `/usr/bin/pip3`
- **base64 链接漏解码**：微信公众号文章常在正文中插入 base64 编码网址（如 `aHR0cHM6Ly9hcS45OS5jb20v...`），入库前必须解码并在文末备注原文，解码命令：`echo "编码字串" | base64 -d`
- **入库前未去重**：每篇文章入库前必须先用 `grep -rl "关键词" /home/baiyun/obsidian-vault/10-Notes/` 确认是否已存在相似主题
- **入库后忘更索引**：新建文章后必须同步更新 `knowledge-index.md`，否则后续无法通过关键词搜索定位文件
- **CloakBrowser 二进制路径**：Linux 下载后缓存在 `~/.cache/cloakbrowser/chrome-linux64/`，无需每次重下
- **CloakBrowser launch 必须在 asyncio 上下文**：必须用 `launch_async()`，不是 `launch()`（后者是 Sync API，会在 asyncio 中报错）
- **用户偏好：直接完成不确认**——抓取完成后直接保存+汇报，不询问"是否要保存"

## 批量采集工作流（>10 篇）

当用户一次发来 N 个微信文章 URL（>10 篇）时，使用以下流程：

### 1. URL 去重 + 规划
先按 URL key（路径中 `/s/` 后的部分）去除重复链接，避免浪费抓取配额。已有本地副本的直接跳过。

### 2. Python 批量抓取（推荐 5~20 篇）

超过 10 篇时不要用 `delegate_task`（URL 在子 agent 间传递容易截断或丢失），直接写 Python 脚本用 `ThreadPoolExecutor` 并发抓取：

```python
import subprocess, re, html as html_module, os, json
from concurrent.futures import ThreadPoolExecutor, as_completed

script = os.path.expanduser("~/.openclaw/workspace/skills/fetch-wx-article/scripts/fetch_wx_article.py")
urls = [(f"{i:02d}", "https://mp.weixin.qq.com/s/xxxxx") for i in range(1, 21)]

def fetch_one(idx_url):
    idx, url = idx_url
    r = subprocess.run(
        ["/usr/bin/python3", script, url],
        capture_output=True, text=True, timeout=60
    )
    return idx, url, r.stdout, r.stderr

with ThreadPoolExecutor(max_workers=4) as executor:
    futures = {executor.submit(fetch_one, u): u for u in urls}
    for future in as_completed(futures):
        idx, url, stdout, stderr = future.result()
        # save to /tmp/articles_batch/{idx}.md
```

**注意**：`max_workers=4` 是安全并发数，避免被微信限流。

### 3. 解析 + 分类

**纯文本解析**（fetch-wx-article.py 已完成 HTML→文本 转换，直接按行处理）：

```python
import re, os

def parse_article(text):
    lines = text.strip().split('\n')
    title = title_idx = ""
    for i, line in enumerate(lines):
        t = line.strip().lstrip('# ').strip()
        if t and not t.startswith('Original') and len(t) > 3 and t not in {'____', '______'}:
            title = t; title_idx = i; break

    noise = {'在小说阅读器读本章','去阅读','在小说阅读器中沉浸阅读',
             'Scan to Follow','继续滑动看下一个','轻触阅读原文',
             'Python爬虫大数据可视化','预览时标签不可点','Got It',
             '____','______','Scan with Weixin','微信扫一扫','Cancel Allow','× 分析','Video Mini Program'}
    content_lines = [l for l in lines[title_idx+1:]
                     if l.strip() not in noise
                     and not l.strip().startswith(('Share Comment','向上滑动','在小说'))]
    content = '\n'.join(content_lines).strip()

    orig_m = re.search(r'^Original\s+(\S+)', text, re.MULTILINE)
    author = orig_m.group(1) if orig_m else ""
    return title, author, content

# 分类映射（按标题关键词）
def classify(title):
    if 'Frida' in title and any(k in title for k in ['加密','算法','Hook','自吐']):
        return "arm-reverse"
    if any(k in title for k in ['ImHex','IDA Pro']):
        return "windows-reverse"
    if any(k in title for k in ['Camoufox','浏览器','Selenium','反检测']):
        return "network-proxy"
    if 'AI' in title and any(k in title for k in ['AST','反混淆']):
        return "ai-reverse"
    if any(k in title for k in ['JS逆向','3DES','MU航司','汇付','支付']):
        return "app-reverse"
    if any(k in title for k in ['FaceX','人脸','WASM']):
        return "ai-agent"
    if any(k in title for k in ['抓包','AI逆向','协议分析','验证码']):
        return "app-reverse"
    return "app-reverse"

# 取下一个可用编号：扫描目录已有文件，取最大编号+1
def next_num(cat_path):
    files = [f for f in os.listdir(cat_path) if f[0].isdigit()]
    nums = sorted([int(f.split('-')[0]) for f in files if f.split('-')[0].isdigit()])
    return (nums[-1] + 1) if nums else 0
```

**URL 去重**：批量处理前，先按 URL key（路径中 `/s/` 后的部分）去重，避免重复抓取已存文章。

**同分类下去重**：先 `ls /home/baiyun/obsidian-vault/10-Notes/<分类>/` 确认无重名，再用 `next_num()` 取最大编号+1。

### 4. 同步索引
入库后必须更新 `knowledge-index.md`：在对应分类章节追加文件条目 + 在关键词索引表格追加行 + 如有新分类则添加分类说明块。

---

*支持文件：*
- `references/docsify-site-to-pdf.md` — docsify 站点转 PDF 的完整工作流（含并发抓取、HTML 转换、Playwright PDF 渲染步骤）
- `references/wx-cli-tool.md` — wx-cli 从源码编译安装指南、GLIBC 兼容性处理、与 fetch-wx-article 的分工对比
- `references/fetch-wx-article.md` — fetch-wx-article.py 工具路径、并发抓取用法、已知失败模式与重试策略（**主用工具**）
- `references/vault-maintenance.md` — 知识库维护：语义搜索、标签、去重、损坏链接巡检
- `references/wechat-browser-access.md` — 浏览器方案访问微信文章的测试记录（代理层拦截 vs 反爬拦截的区分）

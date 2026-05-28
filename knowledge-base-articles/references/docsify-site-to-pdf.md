# docsify 站点转 PDF 工作流

## 背景

用户要求将 `https://hello-agents.datawhale.cc/`（docsify 搭建的教程站）整理为 PDF。

docsify 是纯前端静态站点，playwright 截取首页只有单屏内容，需要**先抓全量 Markdown 章节 → 合并为单个 HTML → 再用 Playwright 转 PDF**。

## 完整流程

### Step 1：并发抓取所有章节

```python
import urllib.parse, concurrent.futures, markdown, subprocess, os

BASE = "https://raw.githubusercontent.com/datawhalechina/Hello-Agents/main/docs/"
CHAPTERS = [
    ("前言", "chapter1/第一章 初识智能体.md"),
    ("第一章", "chapter2/第二章 智能体发展史.md"),
    ...
]

def fetch_chapter(title, path):
    url = BASE + urllib.parse.quote(path)
    for _ in range(3):
        try:
            import urllib.request
            content = urllib.request.urlopen(url, timeout=10).read().decode()
            return title, content
        except:
            pass
    return title, f"## {title}\n\n[获取失败]\n"

with concurrent.futures.ThreadPoolExecutor(8) as ex:
    results = dict(ex.map(lambda p: fetch_chapter(*p), CHAPTERS))
```

### Step 2：合并为单个 Markdown 文件

按顺序拼接各章节 Markdown 内容，拼入主文档。

输出到 `/tmp/<book-name>-book.md`。

### Step 3：Markdown → HTML

```python
html_body = markdown.markdown(
    open('/tmp/book.md').read(),
    extensions=['tables', 'fenced_code', 'toc'],
)
```

HTML 模板要点（避免字体/分页问题）：
```html
<style>
  body { font-family: serif; font-size: 12px; line-height: 1.6; margin: 2cm; }
  h1 { page-break-before: always; }  /* 章节换页 */
  pre { font-size: 10px; page-break-inside: avoid; }
  @page { size: A4; margin: 2cm; }
</style>
```

输出到 `/tmp/<book>.html`。

### Step 4：HTML → PDF（Playwright）

**注意**：直接用 `file://` 协议 playwright 会崩溃，必须通过 HTTP 服务器：

```python
# 启动本地 HTTP 服务器（Python 内置）
proc = subprocess.Popen(
    ['python3', '-m', 'http.server', '8765'],
    cwd='/tmp',  # HTML 文件所在目录
    stdout=subprocess.DEVNULL, stderr=subprocess.DEVNULL
)

# playwright 访问
os.system('playwright pdf "http://localhost:8765/book.html" /tmp/book.pdf')

proc.terminate()
```

### Step 5：复制到目标位置

```bash
cp /tmp/book.pdf /home/baiyun/knowledge/<分类>/<书名>.pdf
```

## Pitfalls

1. **weasyprint 依赖复杂**：`PIL.ImageCms` 和系统 `PIL` 版本冲突，pip 安装后仍报错。放弃使用，直接用 Playwright。
2. **ghostscript 压缩太慢**：50MB PDF 压缩耗时 80s+ 且超时，直接用 playwright 生成的原始 PDF。
3. **file:// 协议不工作**：Playwright 打开 `file:///tmp/...` HTML 会崩溃（60s timeout），必须走 HTTP 服务器。
4. **docsify 页面截取限制**：playwright screenshot/pdf 只能截首页（单屏），无法抓取 docsify 的所有 markdown 内容，必须并发抓取各章节 md 文件再合并。

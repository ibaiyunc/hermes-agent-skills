# 微信文章浏览器访问方案

> 2026-05-28 更新：CloakBrowser + Mihomo 代理可成功打开微信文章，过"环境异常"检测。

> **2026-05-28 结论**：CloakBrowser 是唯一能过微信"环境异常"的浏览器方案。58 处 C++ 层指纹修改 + humanize 行为模拟，原版 Playwright / Playwright + stealth args 均失败。

## 微信"环境异常"检测 vs 代理层拦截 — 两种截然不同的失败模式

| 失败模式 | 症状 | 原因 | 解决方案 |
|----------|------|------|----------|
| **代理层阻断** | `ERR_PROXY_CONNECTION_FAILED`，HTTP 导航失败 | Mihomo TUN fake-ip 劫持了 `mp.weixin.qq.com` 流量 | 关闭系统代理，或 CloakBrowser 走代理下载二进制后直连 |
| **反爬层拦截** | HTTP 200，但页面显示"环境异常" | 微信 JS 检测到自动化特征（webdriver 属性等） | CloakBrowser C++ 层指纹修改 + humanize |

**关键区分**：两种失败都会导致页面打不开，但处理方法完全不同。

## CloakBrowser 完整工作流（唯一有效的浏览器方案）

**核心思路**：CloakBrowser 需要下载 ~200MB 自有 Chromium 二进制（58 处指纹修改）。下载必须走代理。下载完成后，访问微信文章不需要代理。

### Step 1 — 启动 Mihomo 代理

```bash
/opt/clash-party/resources/sidecar/mihomo \
  -d /home/baiyun/.config/mihomo-party/work \
  -f /home/baiyun/.config/mihomo-party/work/config.yaml
# 端口：7890(HTTP), 7891(SOCKS5), 7892(Redir)

# 设置系统代理
gsettings set org.gnome.system.proxy mode 'manual'
gsettings set org.gnome.system.proxy.http host '127.0.0.1'
gsettings set org.gnome.system.proxy.http port 7890
gsettings set org.gnome.system.proxy.https host '127.0.0.1'
gsettings set org.gnome.system.proxy.https port 7890
```

### Step 2 — 下载/安装 CloakBrowser

```bash
/usr/bin/pip3 install cloakbrowser  # 系统 Python，非 hermes venv
```

### Step 3 — 用 launch_async（不是 launch）

CloakBrowser 内部调用 Playwright Sync API，**不能在 asyncio 直接用 `launch()`**，必须用 `launch_async()`：

```python
import asyncio
from cloakbrowser import launch_async

async def fetch_wechat(url):
    browser = await launch_async(
        headless=False,   # headless=True 也可以
        humanize=True,    # 鼠标/键盘/滚动拟人化
        stealth_args=True  # 包含 58 处指纹修改
    )
    page = await browser.new_page()
    resp = await page.goto(url, wait_until="domcontentloaded", timeout=20000)
    await page.wait_for_timeout(3000)
    text = await page.inner_text("body")
    blocked = "环境异常" in text
    if blocked:
        raise RuntimeError("被微信拦截")
    return text
    await browser.close()

result = asyncio.run(fetch_wechat("https://mp.weixin.qq.com/s/xxxxx"))
```

### Step 4 — 关闭系统代理（可选，访问时不需要走代理）

```bash
gsettings set org.gnome.system.proxy mode 'none'
```

## 为什么 Playwright + stealth args 不够用

| 对比项 | Playwright + stealth args | CloakBrowser |
|--------|---------------------------|--------------|
| webdriver 属性 | 通过 CDP 隐藏 | C++ 层直接写成 false |
| 指纹修改 | JS 层面修改，可被检测 | C++ 源码级，58 处全改 |
| humanize | 无 | 鼠标曲线、键盘拟人化、滚动物理节奏 |
| 微信"环境异常" | ❌ 拦截 | ✅ 通过 |
| 下载二进制 | 不需要 | 需要（~200MB，必须走代理） |

## 已测试但失败的方案

| 方案 | 失败原因 |
|------|---------|
| Playwright + CloakBrowser stealth args + 关闭代理 | HTTP 200 但显示"环境异常" |
| CloakBrowser 复用 Playwright Chromium | `launch()` 强制调用 `ensure_binary()` 仍尝试下载 |
| `CLOAKBROWSER_BINARY_PATH` 复用 Playwright Chromium | `launch_async` 内部走 Sync Playwright，在 asyncio 里报错 |

## Mihomo 相关

- **fake-ip filter 问题**：如果 ToDesk 连不上/节点闪现，在 `fake-ip-filter` 加 `+.todesk.com` 后 `kill -USR1 $(pgrep -f mihomo)` 热重载
- **CloakBrowser 二进制路径**：Linux: `~/.cache/cloakbrowser/chrome-linux64/`；Windows/macOS: 自动在对应平台目录
- **无需每次重下**：二进制下载一次后缓存，后续 `launch_async` 直接用

## 快速验证脚本

```python
#!/usr/bin/env python3
"""验证 CloakBrowser 能否打开微信文章"""
import asyncio
from cloakbrowser import launch_async

async def verify():
    url = "https://mp.weixin.qq.com/s/b0Vi_IlccD4VAWk1fNuupA"
    browser = await launch_async(headless=True, humanize=True, stealth_args=True)
    page = await browser.new_page()
    resp = await page.goto(url, wait_until="domcontentloaded", timeout=20000)
    await page.wait_for_timeout(2000)
    text = await page.inner_text("body")
    await browser.close()
    if "环境异常" in text:
        print("❌ 仍被拦截")
    elif resp.status == 200 and len(text) > 100:
        print(f"✅ 成功，HTTP {resp.status}，内容 {len(text)} 字")
    else:
        print(f"? 状态 {resp.status}，内容 {len(text)} 字")

asyncio.run(verify())
```

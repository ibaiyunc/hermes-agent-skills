# wx-cli 安装与使用参考

> 工具：wx-cli（jackwener/wx-cli）
> 用途：本地微信客户端数据 CLI，支持公众号文章、群聊、朋友圈等
> 安装方式：源码编译（npm 包因 GLIBC 版本不兼容不可用）

## 安装（从源码编译）

```bash
# 克隆源码
git clone https://github.com/jackwener/wx-cli.git /tmp/wx-cli

# 编译（需要 Rust）
cd /tmp/wx-cli
cargo build --release

# 安装到系统路径
sudo mv /tmp/wx-cli/target/release/wx /usr/local/bin/wx
wx --version  # wx 0.3.0
```

## 初始化（需要微信正在运行）

```bash
sudo wx init     # 首次初始化，检测数据目录并扫描加密密钥
wx sessions      # 查看最近会话
wx --help        # 查看所有命令
```

## 核心命令

| 命令 | 功能 |
|------|------|
| `wx biz-articles` | 查询公众号文章推送（本地缓存） |
| `wx sessions` | 列出最近会话 |
| `wx history` | 查看聊天记录 |
| `wx search` | 搜索消息 |
| `wx contacts` | 查看联系人 |
| `wx unread` | 显示有未读消息的会话 |
| `wx members` | 查看群成员 |
| `wx stats` | 聊天统计分析 |
| `wx favorites` | 查看微信收藏 |
| `wx sns-feed` | 朋友圈时间线 |
| `wx daemon` | 管理 wx-daemon |

## GLIBC 兼容性问题

预编译的 npm 包（`@jackwener/wx-cli-linux-x64`）需要 GLIBC 2.39，本机（Ubuntu GLIBC 2.35）不兼容，报错：

```
wx: /lib/x86_64-linux-gnu/libc.so.6: version `GLIBC_2.39' not found
```

**解决方案**：从源码编译 Rust 二进制文件，不依赖预编译的 GLIBC 版本。

## Python 依赖安装路径注意

wx-cli 的 Python 依赖（如 scrapling）装到系统 Python 路径：
- 系统 Python：`/usr/bin/python3`（可用）
- hermes venv：`/home/baiyun/.hermes/hermes-agent/venv/bin/python3`（隔离，装了也 `import` 不到）

```bash
# 依赖装到系统 Python
/usr/bin/pip3 install scrapling html2text curl_cffi browserforge
```

## 与 fetch-wx-article 的区别

| 维度 | `wx biz-articles` | `fetch-wx-article`（scrapling）|
|------|-------------------|-------------------------------|
| 数据来源 | 本地微信客户端缓存 | 远程抓取网页 |
| 是否需要微信运行 | **是** | 否 |
| 是否需要登录 | 否（读本地数据） | 否 |
| 公众号范围 | 仅已关注的账号 | 任意公开文章 URL |
| 网络要求 | 不需要 | 需要 |

**结论**：需要本地微信保持登录 → 用 `wx biz-articles`；有公开 URL 但无法访问 → 用 `fetch-wx-article`。

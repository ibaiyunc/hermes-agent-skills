# 知识库维护脚本工具箱

> 基于 2026-05-27 构建的知识库全套维护能力

## 脚本目录

所有脚本位于：`/home/baiyun/scripts/wuanhou/`

**必须用 `/usr/bin/python3`**，不能用 hermes venv 的 python3（没有 chromadb/graphviz）。

| 脚本 | 用途 | 依赖 |
|------|------|------|
| `obsidian-tagger.py` | 批量打标签（自动规则推断） | 无 |
| `knowledge-semantic-search.py` | 纯语义向量检索（nomic embed） | chromadb, ollama |
| `knowledge-hybrid-search.py` | BM25 + 语义混合检索 | chromadb, ollama, rank-bm25 |
| `knowledge-tag-recommend.py` | 基于相似文档聚合推荐标签 | chromadb, ollama |
| `knowledge-summarize.py` | 用标题做 query 提取核心段落做摘要 | chromadb, ollama |
| `knowledge-dedup.py` | 文件名编辑距离 + TF-IDF 内容相似度去重 | 无 |
| `knowledge-broken-links.py` | 检测 wiki/markdown 链接有效性 | 无 |
| `knowledge-tag-check.py` | 标签一致性（大小写、噪音标签） | 无 |
| `knowledge-graph.py` | Graphviz 标签共现谱系图 | graphviz（系统包） |

## 关键 API 陷阱（已踩过的坑）

### Ollama nomic-embed-text 距离度量
```
nomic-embed-text 返回的是欧氏距离（Euclidean），不是余弦相似度。
distance 值越小 → 越相似。

# ✅ 正确（转 0..1 相似度）
sim = 1 / (1 + dist)

# ❌ 错误（会导致相似文档得分极低甚至为负）
sim = 1 - dist
```

### ChromaDB 清空 Collection
```python
# ❌ coll.delete(where={}) 报错
# ValueError: Expected where to have exactly one operator, got {}

# ✅ 正确
client.delete_collection(name="collection_name")
coll = client.get_or_create_collection(name="collection_name")
```

### Graphviz render()
```python
# ❌ 旧版 API
dot.render(pdf_path="/tmp/out")

# ✅ 新版 API
dot.render(filename="/tmp/out", cleanup=True)
```

## 语义索引状态

- 模型：`nomic-embed-text:latest`（Ollama，768维）
- ChromaDB 路径：`~/.hermes/knowledge-base/chroma/`
- Chunk 数：498（49篇文档）
- 增量更新：每 4 小时 cronjob

## Cronjob

| 任务 | 调度 | 交付 |
|------|------|------|
| 知识库定期巡检 | 每周一 10:00 | 微信 |
| 语义搜索增量索引 | 每 4 小时 | 本地 |

## 标签体系（5 维度）

```
工具链:   frida, unidbg, ida, jadx, ast, stalker, burp, ocr, codex, wasm, jni, hook, trace
平台:     android, ios, windows, web, 小程序, arm, linux
类型:     逆向, 抓包, 反混淆, 反调试, 教程, 速查, 实战, 笔记, 工具推荐, 清单, 发布, 入门, 进阶, 高级, 免杀, 算法
专题:     ctf, 量子计算, cloudflare, tailscale, js逆向, claude, ssl-pinning, algokiller, spiderbox, aicryptoproxy, mcp
元:       原创, 翻译, 整理
```

## 使用示例

```bash
# 语义检索
/usr/bin/python3 scripts/wuanhou/knowledge-hybrid-search.py --query "frida hook native函数" --top 5

# 批量打标签（预览）
/usr/bin/python3 scripts/wuanhou/obsidian-tagger.py --dry
# 写入
/usr/bin/python3 scripts/wuanhou/obsidian-tagger.py

# 标签推荐（预览）
/usr/bin/python3 scripts/wuanhou/knowledge-tag-recommend.py --file "arm-reverse/tutorials/frida-native-hook.md"
# 写入 frontmatter
/usr/bin/python3 scripts/wuanhou/knowledge-tag-recommend.py --file "path/to/doc.md" --apply

# 智能摘要（预览）
/usr/bin/python3 scripts/wuanhou/knowledge-summarize.py --file "path/to/doc.md"
# 写入
/usr/bin/python3 scripts/wuanhou/knowledge-summarize.py --file "path/to/doc.md" --apply

# 去重检测
/usr/bin/python3 scripts/wuanhou/knowledge-dedup.py

# 损坏链接
/usr/bin/python3 scripts/wuanhou/knowledge-broken-links.py

# 标签一致性
/usr/bin/python3 scripts/wuanhou/knowledge-tag-check.py

# 知识谱系图
/usr/bin/python3 scripts/wuanhou/knowledge-graph.py
# 输出: /home/baiyun/obsidian-vault/.assets/knowledge-graph.png
```

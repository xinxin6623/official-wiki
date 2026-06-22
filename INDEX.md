# Official-wiki

微信学校公号 → Vault → 简报 流水线。借鉴 obwiki 构建原则做**双轨沉淀**（wiki concept + LightRAG）+ **双轨检索**（逻辑走 wiki、精准走 RAG）。

> 🤖 Agent 上手先读 [`AGENTS.md`](./AGENTS.md) 的操作守则（通用协议在 [`docs/trio-protocol.md`](./docs/trio-protocol.md)）；改动后记得追加 [`CHANGELOG.md`](./CHANGELOG.md)（强标签格式见文件顶部）。进度走 CHANGELOG + 下方「当前接力点」。

## 当前接力点 (Handoff)

> 只保留最新一条，下一轮 wrap-up 直接覆盖。历史接力沉淀到 obwiki。

### 概述
**Phase 4 已完成；Phase 5 已完成切片/staging 与本机 LightRAG 安装，但真实灌库被本地 `qwen3:14b` 抽取超时卡住。下一步优先做两件事：给 LightRAG 换更快/远端 LLM 跑通真实索引；把 Docker `wx-exporter` 接成新文章 URL 发现层。**
**⚠ 阻塞：`localhost:3000` 的 `wx-exporter` 是当前新发文章发现依赖；`fetch.py` 只能消费 `sources.yaml` 已知 URL，学校不变也不能自己发现新发布内容。**

### 明细
- **2026-06-21 最新状态**：
  - ✅ Phase 4 `~/projects/wx-brief/distill.py` 首版落地：支持 `--school` / `--since` / `--limit` / `--force` / `--dry-run`，把 `distilled:false` raw 归入每校 `wiki/concepts/`，生成 `wiki/graph.json`，并翻 raw frontmatter 状态位。
  - ✅ 当前 vault：`~/.ai-vault/wechat/` 共 10 校 / 42 篇 raw / 29 个 concept / 10 个 graph；`distilled_raw=42/42`，重跑 dry-run 显示 0 待处理。
  - ✅ 当前 sources：`~/projects/wx-brief/sources.yaml` 共 10 所海淀学校 / 98 条 URL；每日 03:20 定时任务已在 2026-06-21 净增 9 篇。
  - 🚧 Phase 5 `~/projects/wx-brief/index_rag.py` 首版落地：默认 `staging` 后端已验证 2 篇 raw → 7 chunks，写 `state/rag_staging.jsonl`，幂等重跑 0 增量；staging 不翻 `rag_indexed`。
  - 🚧 LightRAG 已装进 `~/projects/wx-brief/.venv`，本机服务脚本为 `~/projects/wx-brief/bin/lightrag_server_local.sh`；embedding 用 `nomic-embed-text:latest`，`EMBEDDING_DIM=768`。真实文章入库时卡在本地 `qwen3:14b` entity extraction 超时，LightRAG/Ollama 已按用户要求停止，端口 `9621`/`11434` 当前未监听。
  - ✅ Docker `wx-exporter` 正在运行：容器 `wx-exporter`，镜像 `ghcr.io/wechat-article/wechat-article-exporter:latest`，端口 `3000`；`http://localhost:3000/dashboard/account` 属于它。若要抓取固定学校的新发布文章，必须用它或等价发现层导出 URL，再回填 `sources.yaml`。
  - **下一步建议**：先实现 `wx-exporter` → `sources.yaml` 的 URL 同步/去重脚本；并把 LightRAG 的 LLM 抽取改到更快/远端模型后，重跑 `index_rag.py --backend lightrag-api --endpoint http://127.0.0.1:9621 --limit 1` 验证真实索引。
- **2026-06-20 状态**：
  - ✅ `~/projects/wx-brief/fetch.py` 已支持安全扩抓参数：`--limit` / `--per-school-limit` / `--sleep-min` / `--sleep-max` / `--cooldown-every` / `--cooldown-seconds`；失败写 `state/failures.json`，每篇即时保存 ledger。
  - ✅ Obsidian 兼容性修正完成：总 `INDEX.md` 与学校 `INDEX.md` 表格内用 Markdown 相对链接；`wiki/index.md` 与 `wiki/entities/` 保留 Obsidian wikilink；raw 只做证据层，不硬插双链。
  - ✅ 当前 vault：`~/.ai-vault/wechat/` 共 3 校 / 12 篇；每校 4 篇；全库校验 `missing_doc_links=0` / `bad_table_rows=0` / `missing_or_empty_images=0` / `noise_files=0`。
  - ✅ 安全扩抓实测命令已跑通：`cd ~/projects/wx-brief && .venv/bin/python fetch.py --limit 6 --per-school-limit 2 --sleep-min 8 --sleep-max 15 --cooldown-every 6 --cooldown-seconds 45`。
  - ✅ 已安装定时抓取：`~/Library/LaunchAgents/com.james.wx-brief.fetch.plist` → `~/projects/wx-brief/bin/wx_brief_fetch_daily.sh`；每天 `03:20` 跑一轮，每轮最多 9 篇、每校最多 3 篇、篇间 45–120s、每 3 篇冷却 300s。
  - **下一步建议**：持续用 exporter/人工把海淀各学校近一年 URL 低频回填进 `sources.yaml`；定时任务只消费已有 URL，不自动发现新学校/新文章。
  - **Phase 4 接力**：近期 raw 入库稳定后，实现 `distill.py`，把 `raw/` 蒸馏到每校 `wiki/concepts/` 与 `wiki/entities/`，按 obwiki 渐进式披露补 L1 和双链。✅ 2026-06-21 已完成首版。
- **关键文件**：脚本工程 `~/projects/wx-brief/`；vault `~/.ai-vault/wechat/`；项目方案 `wx-brief-implementation-plan.md`；本轮记录见 `CHANGELOG.md` 2026-06-20 多条。

## 项目结构

```
Official-wiki/
├── AGENTS.md                          # agent 操作守则（本项目专属硬规则）
├── INDEX.md                           # 本文件：导航 + 接力点
├── CHANGELOG.md                       # 强标签演绎记录
├── wx-brief-implementation-plan.md    # 实施方案（开发 single source of truth）
└── docs/
    └── trio-protocol.md               # 三件套通用协议（standard-v3）
```

> **实际落地路径**（机器侧，不在本仓库）：脚本工程 `~/projects/wx-brief/`（fetch/distill/index_rag/retrieve.py + sources.yaml + state/）；入库 vault `~/.ai-vault/wechat/`（`schools/<区>-<校>/raw/` 原文层 + `schools/<区>-<校>/wiki/` 逻辑层 + 总 `INDEX.md` L0 索引）。

## 子模块导航

| 文件 | 用途 |
|---|---|
| [`wx-brief-implementation-plan.md`](./wx-brief-implementation-plan.md) | 实施方案：架构 / 七个 Phase / frontmatter schema / 合规 / 验收 checklist |
| [`docs/trio-protocol.md`](./docs/trio-protocol.md) | 三件套通用协议 |

## 常用操作

```bash
# Phase 1：装转换引擎（Mac mini）
uv tool install wechat-article-to-markdown
# 抓取（落 raw）
cd ~/projects/wx-brief && python fetch.py
python fetch.py --school "海淀-北京市海淀区第二实验小学"
# 沉淀双轨
python distill.py        # 轨 A：raw → wiki concept
python index_rag.py      # 轨 B：raw → LightRAG
# 检索分流（逻辑→wiki / 精准→RAG）
python retrieve.py "XX中学今年的德育主线是什么"
python retrieve.py --raw "X月X日运动会的具体项目安排"
```

## 相关链接

- 📓 演绎记录 / 进度：[CHANGELOG.md](./CHANGELOG.md)
- 🤖 Agent 守则：[AGENTS.md](./AGENTS.md)
- 🧠 借鉴来源：[obwiki skill](../myskills/obwiki/)
<!-- 在此补充：仓库地址、部署地址、设计文档、issue tracker 等 -->

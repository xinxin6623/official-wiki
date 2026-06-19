# Official-wiki

微信学校公号 → Vault → 简报 流水线。借鉴 obwiki 构建原则做**双轨沉淀**（wiki concept + LightRAG）+ **双轨检索**（逻辑走 wiki、精准走 RAG）。

> 🤖 Agent 上手先读 [`AGENTS.md`](./AGENTS.md) 的操作守则（通用协议在 [`docs/trio-protocol.md`](./docs/trio-protocol.md)）；改动后记得追加 [`CHANGELOG.md`](./CHANGELOG.md)（强标签格式见文件顶部）。进度走 CHANGELOG + 下方「当前接力点」。

## 当前接力点 (Handoff)

> 只保留最新一条，下一轮 wrap-up 直接覆盖。历史接力沉淀到 obwiki。

### 概述
**Phase 1 已真机跑通（引擎装好、内核就绪、单篇抓取验证）。下一步 Phase 2：写 `sources.yaml`，然后 Phase 3 写 `fetch.py`（按实测的"产物落包目录"坑实现产物定位）。**

### 明细
- **2026-06-17**：
  - ✅ **Phase 1 完成**：`wechat-article-to-markdown` v0.1.0 装于 `~/.local/bin/`；Camoufox 内核 v135.0.1-beta.24 经 `gh-proxy.com` 镜像下载并解压到 `~/Library/Caches/camoufox`（`camoufox version` = Up to date）；单篇真机抓取通过（北京二小一篇，32/33 图片本地化，元信息齐全）。
  - ⚠ **实测关键坑**：产物落**包安装目录** `.../site-packages/output/`，**不是 CWD**，`cd` 无效。`fetch.py` 须 diff 包目录定位产物并搬回 `.raw/output/`（方案 Phase 1 已知坑#1 + Phase 3 `fetch_one` 已按实情改）。
  - **下一步**：Phase 2 `sources.yaml` → Phase 3 `fetch.py`（落 raw + 三状态位 frontmatter）→ Phase 4 distill（轨 A）/ Phase 5 index_rag（轨 B，复用现有 LightRAG）/ Phase 6 retrieve（分流）/ Phase 7 `/wx-brief`。
- **2026-06-18**：依赖与内核全量备份到 `vendor/`（zip 284M + 内核 611M + venv 302M，含 `README.md` 恢复指南，`.gitignore` 已忽略）。换机恢复走方式 A（只铺内核 + `uv tool install` 重装 CLI），别直接铺 venv（绝对路径陷阱）。
- 脚本工程 `~/projects/wx-brief/`（已建 `state/` `.raw/`），vault `~/.ai-vault/wechat/`。关键决策：RAG = 复用现有 LightRAG；wiki 沉淀 = 独立 distill 步骤。

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

> **实际落地路径**（机器侧，不在本仓库）：脚本工程 `~/projects/wx-brief/`（fetch/distill/index_rag/retrieve.py + sources.yaml + state/）；入库 vault `~/.ai-vault/wechat/`（`raw/` 原文层 + `wiki/` 逻辑层 + `INDEX.md` L0 索引）。

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

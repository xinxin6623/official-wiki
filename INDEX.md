# Official-wiki

微信学校公号 → Vault → 简报 流水线。借鉴 obwiki 构建原则做**双轨沉淀**（wiki concept + LightRAG）+ **双轨检索**（逻辑走 wiki、精准走 RAG）。

> 🤖 Agent 上手先读 [`AGENTS.md`](./AGENTS.md) 的操作守则（通用协议在 [`docs/trio-protocol.md`](./docs/trio-protocol.md)）；改动后记得追加 [`CHANGELOG.md`](./CHANGELOG.md)（强标签格式见文件顶部）。进度走 CHANGELOG + 下方「当前接力点」。

## 当前接力点 (Handoff)

> 只保留最新一条，下一轮 wrap-up 直接覆盖。历史接力沉淀到 obwiki。

### 概述
**Phase 1/2/3 已真机跑通：引擎+内核就绪、`sources.yaml` 解析、`fetch.py` 抓取入库落 raw（含三状态位 frontmatter、自动 L0 INDEX、幂等去重）。下一步 Phase 4：写 `distill.py`（轨 A·raw → wiki concept + 建链，走 LiteLLM `local-qwen3-14b`）。**

### 明细
- **2026-06-20**：
  - ✅ **Phase 2/3 完成**（`~/projects/wx-brief/`）：`sources.yaml`（URL 唯一人工入口）+ `fetch.py`（抓取→frontmatter→content-hash 去重→落 `~/.ai-vault/wechat/raw/`→重建 L0 `INDEX.md`）+ `pyproject.toml` + `README.md`。
  - **真机实测**：昌平二小招生公告一篇，9/9 图落 `.assets/`、正文路径已改写、发布日/公号名从文章头部 `> 公众号:` 自动解析；二次运行净增=0（幂等）。
  - **实现要点**：CLI 走绝对路径 `~/.local/bin/`；产物 `output/` 用 diff 包目录定位（坑#1）；退出码不可靠靠产物判成功（坑#2）；URL 间 2–5s 随机 sleep。
  - **下一步**：Phase 4 `distill.py`（轨 A）→ Phase 5 `index_rag.py`（轨 B，复用现有 LightRAG）→ Phase 6 `retrieve.py`（分流）→ Phase 7 `/wx-brief`。
- **2026-06-17**：✅ **Phase 1**：`wechat-article-to-markdown` v0.1.0 装于 `~/.local/bin/`；Camoufox 内核 v135.0.1-beta.24 经 `gh-proxy.com` 镜像装好（`camoufox version`=Up to date）。⚠ 关键坑：产物落**包安装目录** `site-packages/output/` 非 CWD（已在 fetch.py 用 diff 法处理）。
- **2026-06-18**：依赖与内核全量备份到 `vendor/`（zip 284M + 内核 611M + venv 302M，`.gitignore` 已忽略）。换机恢复走方式 A（铺内核 + `uv tool install` 重装 CLI），别直接铺 venv（绝对路径陷阱）。
- vault `~/.ai-vault/wechat/`（`raw/` + `wiki/concepts|entities/` + 自动 `INDEX.md`）。关键决策：RAG = 复用现有 LightRAG；wiki 沉淀 = 独立 distill 步骤。

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

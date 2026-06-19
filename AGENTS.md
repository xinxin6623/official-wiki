---
trio: standard-v3
trio-initialized: 2026-06-17
status: active
desc: "微信学校公号→双轨沉淀→简报流水线"
---

<!--
status / desc 是 TaskBoard 看板字段（接入看板才用，不接入可删）：
  status: active | paused | done —— 看板卡片状态徽标与排序
  desc:   ≤30 字一句话项目描述 —— 看板卡片首页显示，空则不显示
看板完整契约（阶段表/简介/架构图/接力点）见 TaskBoard 项目 02 §1.1b；一键产出全套用 /outkanban。
-->

# Official-wiki · Agent 操作守则

> **上来先读这份**，再看 [`INDEX.md`](./INDEX.md) 找模块和导航。
>
> **通用三件套协议**见 [`docs/trio-protocol.md`](./docs/trio-protocol.md)（文档维护节奏 / Handoff 写入 / 子项目嵌套 / 记忆三条线边界 / 语言规则 / 跨项目反例）。**本文件只列本项目专属守则**。
>
> **`trio: standard-v3`** = 本项目按当前标准维护三件套。其他 agent / skill 可通过本行 frontmatter 判断是否按标准流程处理本项目。

## 这是什么项目

微信学校公号 → Vault → 简报 的自动化流水线。借鉴 [obwiki](../myskills/obwiki/) 的知识库构建原则（渐进式披露 L0→L1→L2、链接密度为北极星、ADD/UPDATE/MERGE/NOOP 入库判定、查询即回写），把单向"抓取→简报"升级为**双轨沉淀 + 双轨检索**：

- **沉淀双轨**：raw 原文 →（轨 A）蒸馏成 wiki concept 页（逻辑层）+（轨 B）灌进 LightRAG 向量库（原文层）。
- **检索双轨**：**逻辑/概念检索先走 wiki**（concept + 链接图谱，快且省 token）；**精准/取证检索走 LightRAG**（向量召回原文片段）。
- 简报生成是这套双轨的消费者，不是流水线终点。

当前阶段：**方案已定稿，进入连续开发**。实施方案见 [`wx-brief-implementation-plan.md`](./wx-brief-implementation-plan.md)。

## 上手三步

1. 读 [`INDEX.md`](./INDEX.md)，看项目结构、子模块导航和当前接力点。
2. 读 [`wx-brief-implementation-plan.md`](./wx-brief-implementation-plan.md)——它是开发的 single source of truth（架构 / 七个 Phase / frontmatter schema / 验收锚点）。
3. 看 Phase 顺序：fetch（落 raw）→ distill（轨 A·wiki）→ index_rag（轨 B·RAG）→ retrieve（检索分流）→ wx-brief（简报）。

## 项目专属硬规则

> 通用守则（语言 / 节奏 / Handoff / 子项目 / 记忆边界）见 `docs/trio-protocol.md`。本段只列**本项目专属**约束。

- **raw 原文不可变**：`~/.ai-vault/wechat/raw/` 落地后只读。蒸馏 / 切片 / 综合都在下游做，绝不回改 raw 正文。
- **三轨各自幂等**：`fetch`（content-hash 去重）/ `distill`（`distilled` 状态位）/ `index_rag`（`rag_indexed` 状态位）独立翻转，重跑只补未完成的，不重复处理。
- **沉淀不重复存正文**：同一份 raw 喂 wiki 轨与 RAG 轨；wiki 存提炼后的 concept，RAG 存切片，**正文只在 raw/ 留一份**。
- **链接密度是北极星**：distill 默认改已有 concept 页而非新建；每条 `[[链接]]` 必须带"为什么相关"，禁裸链。
- **人在环**：DELETE 永不自动；concept 页冲突标 `needs_review` 出清单给人确认，不自动消解。
- **检索默认先走 wiki**：逻辑题走 wiki 轨（L0→L1→反思→按需 L2/RAG）；只有要原话/具体数字/取证才走 RAG 轨（`--raw`）。
- **复用现有 LightRAG**：RAG 轨灌进已有 LightRAG 实例，不另起一套；文档 id = content_hash 保证可重灌。
- **模型走 LiteLLM 别名**：蒸馏/摘要用 `local-qwen3-14b`，跨篇综合用 `claude`，不硬编码 endpoint。
- **不要随手改 `.env` / 凭证 / `settings.json`**：敏感配置由项目所有者维护。
- **不要主动删除文件**：废弃 → 移到 `archive/` 或 `不加载/`，不要 `rm`。

<!-- 在此追加本项目工作中沉淀的专属硬规则（活文档，随时更新） -->

## 目录命名约定

| 子目录 | 用途 |
|---|---|
| `docs/` | 详细文档（根项目持有 `trio-protocol.md`） |
| `~/projects/wx-brief/` | 脚本工程（fetch/distill/index_rag/retrieve.py + sources.yaml + state/）——见方案 §2 |
| `~/.ai-vault/wechat/` | 入库目标 vault：`raw/`（原文层）+ `wiki/`（逻辑层）+ `INDEX.md`（L0） |
| `archive/` 或 `不加载/` | 归档区 |

> 注意：脚本与 vault 实际落在 `~/projects/wx-brief/` 与 `~/.ai-vault/wechat/`（机器路径），本仓库（`Official-wiki/`）持有**方案与三件套文档**。开发时两处对照看。

## 项目专属"不要做的事"

> 通用反例（错配三件套节奏 / 误记父级 CHANGELOG / 把项目状态塞 auto memory 等）见 `docs/trio-protocol.md` §9。本段只列**本项目专属**反例。

- ❌ 引入公号登录依赖（exporter 是可选模块，默认不启用——见方案 §6）
- ❌ 把 wiki 轨与 RAG 轨混成一套检索（两轨正交：wiki 给地图，RAG 给现场证据）
- ❌ 在抓取层做裁剪 / 蒸馏（raw-first，加工全在下游）
- ❌ 并发轰公号接口（低频跑，URL 间 2–5s 随机 sleep）
- ❌ 替用户做 `git push --force` / 任何不可逆操作（必须先问）

<!-- 在此追加本项目工作中沉淀的专属反例 -->

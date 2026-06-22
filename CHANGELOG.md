# Official-wiki · CHANGELOG

> 每次动了什么记一条。详细记录写在各自模块目录下，根目录 CHANGELOG 是**强标签化的检索索引**。
>
> **如本项目下有子项目**（子目录里也有 AGENTS/INDEX/CHANGELOG 三件套）：本 CHANGELOG **只记录跨多个子项目的同时操作**；单一子项目操作记在该子项目自己的 CHANGELOG 里。详见 [`docs/trio-protocol.md`](./docs/trio-protocol.md) §5 子项目嵌套（max-depth = 3）。

## 格式规范（严格）

```
## YYYY-MM-DD #<type> scope:<name> [#<extra-tag>...] - <一句话主题>

- Why: <一句话动机，不复述 what>
- 详见: <path 或 commit hash>
```

**硬约束**：
- 日期必须 ISO 格式 `YYYY-MM-DD`
- 类型标签必须以 `#` 开头，从下面字典选一个为主标签
- 作用域必须 `scope:<name>` 形式，name 用 kebab-case；多模块改动用多个 `scope:`
- Why 一行不超过 80 字符
- **不贴 diff、不复述 what**——那些进 commit 或模块自己的文档

## 类型标签字典

| 标签 | 含义 |
|---|---|
| `#feat` | 新功能 |
| `#fix` | bug 修复 |
| `#refactor` | 重构（无行为变化） |
| `#perf` | 性能优化 |
| `#docs` | 文档变更 |
| `#test` | 测试相关 |
| `#chore` | 构建/依赖/工具链/初始化 |
| `#archive` | 归档/弃用 |
| `#breaking` | 破坏性变更（叠加） |
| `#deprecated` | 标记弃用（叠加） |
| `#wip` | 进行中（叠加） |

## 检索示例

```bash
grep -E "^## .* #feat .* scope:fetch" CHANGELOG.md    # fetch 模块新功能
grep "#breaking" CHANGELOG.md                          # 所有破坏性变更
grep "^## 2026-06" CHANGELOG.md                        # 2026 年 6 月所有动作
```

---

## 2026-06-22 #feat scope:sources scope:fetch - 批量补入学校公号并抓取近期文章

- Why: 名单校名需按官方/规范名称落库,每校至少有近期 raw 文章供后续蒸馏
- 详见: 学校公众号抓取入库执行报告.csv + ~/projects/wx-brief/sources.yaml

## 2026-06-21 #chore scope:lightrag #wip - 安装 LightRAG 并定位本地抽取超时

- Why: RAG 轨需真实 endpoint 验证,但本地模型抽取性能决定能否稳定灌库
- 详见: ~/projects/wx-brief/bin/lightrag_server_local.sh（qwen3:14b 抽取超时;服务已停）

## 2026-06-21 #docs scope:exporter scope:sources - 明确 wx-exporter 是新文章发现依赖

- Why: 固定学校也需要发现新 URL,否则 fetch 只能消费 sources.yaml 已知链接
- 详见: wx-brief-implementation-plan.md §6 + http://localhost:3000/dashboard/account

## 2026-06-21 #feat scope:distill scope:vault - Phase 4 首版蒸馏到 wiki concepts

- Why: raw 入库后需要 L1 逻辑层承接默认 wiki 检索,为后续简报与 RAG 分流铺路
- 详见: ~/projects/wx-brief/distill.py（42 篇 raw → 29 个 concept + 10 个 graph）

## 2026-06-21 #feat scope:index-rag #wip - Phase 5 切片与 staging 后端落地

- Why: 真实 LightRAG 接入前需先固化切片、metadata 与幂等边界,避免误翻状态位
- 详见: ~/projects/wx-brief/index_rag.py（staging 2 篇 → 7 chunks,真实 endpoint 待接）

## 2026-06-20 #chore scope:fetch #perf - 安装低频定时抓取任务

- Why: 近一年文章回填需长期慢速执行,避免一次性高频抓取触发访问限制
- 详见: ~/Library/LaunchAgents/com.james.wx-brief.fetch.plist + ~/projects/wx-brief/bin/wx_brief_fetch_daily.sh

## 2026-06-20 #feat scope:fetch #perf - 增加限速分批抓取并完成 6 篇扩抓

- Why: 扩大学校公号采集需避免访问限制,必须串行限速、可恢复、可验证
- 详见: ~/projects/wx-brief/fetch.py（--limit/--per-school-limit/sleep/cooldown;当前 3 校 12 篇）

## 2026-06-20 #feat scope:vault scope:obsidian - 补齐学校 wiki 骨架与 Obsidian 链接

- Why: raw 归档需与 L1 wiki 层分离,让 INDEX 真正成为可点击的渐进式披露入口
- 详见: ~/projects/wx-brief/fetch.py（3 校各再采样 1 篇;全库 wikilink/图片校验通过）

## 2026-06-20 #fix scope:obsidian - 修复 INDEX 表格链接被 wikilink alias 打断

- Why: Obsidian 表格会把 `[[path|alias]]` 的竖线当分列,需改为 Markdown 相对链接
- 详见: ~/projects/wx-brief/fetch.py（全库 missing_links=0,bad_table_rows=0）

## 2026-06-20 #feat scope:sources scope:fetch - 接入 exporter URL 并验证单校抓取

- Why: 真实学校公号链接需进入 sources.yaml,才能从方案转入批量入库验证
- 详见: ~/projects/wx-brief/sources.yaml（3 校 77 条 URL;三校各试抓 1 篇成功）

## 2026-06-20 #feat scope:fetch - Phase 2/3 落地:sources.yaml + fetch.py 真机跑通

- Why: 抓取层须把原文不可变落 raw,后续双轨沉淀才有料;raw-first 闭环第一环
- 详见: ~/projects/wx-brief/{sources.yaml,fetch.py,README.md}（昌平二小招生公告实测,9/9 图,幂等净增=0）

## 2026-06-19 #chore scope:repo - 建本地 git 并推 GitHub 远端同步

- Why: 文档此前纯本地无版本/无备份,接入远端便于换机与协作
- 详见: github.com/xinxin6623/official-wiki (PUBLIC,仅同步文档;vendor/.raw 已 gitignore)

## 2026-06-18 #chore scope:vendor - 全量备份转换引擎依赖与 Camoufox 内核

- Why: 内核镜像下载慢易断,离线留底便于换机/重装一键恢复
- 详见: vendor/README.md（内核 zip 284M + 解压内核 611M + uv venv 317M）

## 2026-06-17 #feat scope:fetch #wip - Phase 1 转换引擎装好并真机跑通

- Why: 抓取层是整条流水线起点,需先验证 to-markdown + Camoufox 真机可用
- 详见: wx-brief-implementation-plan.md Phase 1（含实测修正的已知坑）

## 2026-06-17 #fix scope:plan - 修正输出落点坑:产物落包目录非 CWD

- Why: 真机实测推翻"落 CWD/output"假设,fetch.py 定位逻辑须按实情改
- 详见: wx-brief-implementation-plan.md Phase 1 已知坑#1 + Phase 3 fetch_one

## 2026-06-17 #docs scope:plan - 实施方案升级为双轨沉淀+双轨检索

- Why: 借鉴 obwiki 构建原则，逻辑检索走 wiki、精准检索走沉淀 RAG，并保留 RAG 系统
- 详见: wx-brief-implementation-plan.md（§1 架构 / Phase 4-6 新增 / §4 schema 三状态位）

## 2026-06-17 #chore scope:init - 项目初始化

- Why: 新项目需要 agent 入口、人类导航、演绎记录三件套，便于 agent 协作和未来 LLM 检索
- 详见: AGENTS.md / INDEX.md / 本文件

<!-- 新条目加在这里上方，保持最新在最上 -->

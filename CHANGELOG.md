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

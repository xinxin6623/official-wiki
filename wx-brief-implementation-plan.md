# 微信学校公号 → Vault → 简报 流水线 · 实施方案

> 交付对象:Claude Code Agent 执行
> 目标环境:Mac mini M4(常驻服务器)+ MacBook,macOS,已有 `~/.ai-vault/` / LiteLLM / Langfuse / LightRAG / Claude Code 生态
> 文档约定:重要逻辑的代码注释用中文(给 James 读),内联辅助注释用英文(给 AI 参考)

> **本版核心(2026-06 优化)**:借鉴 [obwiki](../../myskills/obwiki/) 的知识库构建原则(渐进式披露 L0→L1→L2、链接密度为北极星、ADD/UPDATE/MERGE/NOOP 入库判定、查询即回写),把单向"抓取→简报"流水线升级为**双轨沉淀 + 双轨检索**:
> - **沉淀双轨**:raw 原文 → ① 蒸馏成 wiki concept 页(逻辑层) + ② 灌进 LightRAG 向量库(原文层)。
> - **检索双轨**:**逻辑/概念检索先走 wiki**(concept 页 + 链接图谱,渐进式披露,快且准);**精准/取证检索走 LightRAG**(向量召回原文片段,要原话、要细节时用)。
> - 简报生成是这套双轨的一个消费者,不再是流水线终点。

---

## 0. 选型结论(已核实,2026-06)

| 工具 | 角色 | 关键事实 | 是否需登录 |
| :--- | :--- | :--- | :--- |
| `jackwener/wechat-article-to-markdown` | **主干 · 转换引擎** | 628★,活跃维护;Camoufox 反指纹;原生 Claude Code skill;单篇 URL→纯净 MD + 图片本地化 | **否** |
| `wechat-article-exporter` | **可选 · 发现 + KPI** | 7.3k★,很活跃;走官方超链接接口;能导阅读量/评论/Excel;Docker/Cloudflare 私有化 | **是(需公号扫码)** |
| `xiaoguyu/wechatDownload` | 弃用 | 已归档 + Windows GUI,与本环境不符 | — |

**决策**:主链路用 `to-markdown` 做转换引擎,**不引入公号登录依赖**。发现层对低频场景用人工维护的 `sources.yaml` 链接清单(最稳、零维护)。仅当确实需要"阅读量/评论 KPI 简报"或"某公号全量历史回填"时,才另起 exporter 可选模块(见 §6)。

**关键认知**:`to-markdown` 不做文章发现,只转换已知 URL。不要把它当成"输入公号名就自动抓全号"的工具——那是 exporter 的能力,但代价是公号登录。

---

## 1. 架构(沉淀双轨 + 检索双轨)

```
[采集层] sources.yaml(公号 + 文章 URL 清单,人工/exporter 产出)
    ↓
[抓取层] to-markdown CLI 批量转换 → 原始 MD + images/
    ↓
[入库层] 加 frontmatter + content-hash 去重 → raw 原文原子落入 ~/.ai-vault/wechat/schools/<区>-<校>/raw/
    │      ↑ 借 obwiki:raw-first、四操作判定(ADD/UPDATE/MERGE/NOOP)、链接密度
    ├──────────────────────────────┬──────────────────────────────┐
    ↓ (沉淀轨 A · 逻辑层)            ↓ (沉淀轨 B · 原文层)
[蒸馏层] distill:raw → concept   [索引层] index-rag:raw 切片
   页(200-400字)+ 实体/链接          → 灌入 LightRAG 向量库
   → schools/<区>-<校>/wiki/          → 精准/取证检索的语料源
    │                                  │
    └──────────────┬───────────────────┘
                   ↓
[检索分流] route:逻辑/概念问题 → wiki(L0→L1→按需 L2,渐进式披露)
                  精准/取证问题 → LightRAG(向量召回原文片段)
                   ↓
[简报层] /wx-brief → 逐篇摘要(local-qwen3-14b)+ 跨篇综合(claude)
   消费 wiki concept 做主题聚类,按需向 RAG 取原文佐证 → 简报 MD
```

设计原则(沿用你的既有约定 + 注入 obwiki 理念):
- **raw-first**:抓取层只存原文,不裁剪;加工(蒸馏/切片/综合)都在下游做。raw 不可变。
- **沉淀双轨各司其职**:wiki concept 页负责"是什么/逻辑关系"(逻辑层);LightRAG 负责"原话/细节/取证"(原文层)。同一份 raw 喂两轨,不重复存正文。
- **检索双轨分流(本版关键)**:**逻辑检索先走 wiki**——concept 页 + 链接图谱已把多篇文章的逻辑结构沉淀好,渐进式披露快且省 token;**精准检索走 RAG**——要原文片段、具体数字、政策原话时,走 LightRAG 向量召回。两轨正交:wiki 给"地图",RAG 给"现场证据"。
- **链接密度是北极星**(obwiki 第 3 铁律):distill 默认改已有 concept 页而非新建;每条 `[[链接]]` 带"为什么相关"。学校公号天然高互链(同校多文、政策连续、人物复现)。
- **入库四操作**(obwiki 第 4 铁律):每篇 raw 判定 ADD/UPDATE/MERGE/NOOP;`DELETE` 只提议不自动;冲突标记给人看。
- **content-hash 去重**:复用 WeChat 监控系统的内容哈希去重,避免重复抓取/重复入库/重复索引。
- **LiteLLM 统一路由**:蒸馏层与简报层所有模型调用走 `models.yaml` 别名,不硬编码。

---

## 2. 目录结构

```
~/projects/wx-brief/
├── sources.yaml              # 采集清单(唯一需人工维护的文件)
├── pyproject.toml            # uv 管理,依赖 to-markdown + lightrag 客户端
├── fetch.py                  # 抓取层 + 入库层 orchestrator(落 raw)
├── distill.py                # 沉淀轨 A:raw → concept 页 + 链接(独立步骤,可批量)
├── index_rag.py             # 沉淀轨 B:raw 切片 → 灌 LightRAG
├── retrieve.py              # 检索分流:逻辑→wiki / 精准→RAG(供 /wx-brief 与人工查询共用)
├── state/
│   ├── seen_hashes.json      # content-hash 去重账本
│   └── rag_indexed.json      # 已灌 RAG 的文件账本(避免重复索引)
├── .raw/                     # to-markdown 的临时 output/(每轮清空)
└── README.md

~/.ai-vault/wechat/           # 入库总目标(vault 内,对齐 obwiki 布局)
├── INDEX.md                  # 全市总览 L0:各校库导航 + 跨校汇总概览
└── schools/                  # 物理分库:每所学校一个独立子库
    └── <区>-<学校名>/         # 例:海淀-中关村第一小学
        ├── INDEX.md          # 本校 L0 索引层:本校全文 frontmatter 概览
        ├── meta.yaml         # 本校元信息(公号名/biz/区/学段/抓取状态)
        ├── raw/              # 本校 raw 原文(不可变,RAG 语料源)
        │   └── <YYYY-MM-DD>__<account>__<slug>.md
        └── wiki/             # 本校沉淀轨 A:逻辑层
            ├── concepts/     # concept 页(200-400字,karpathy 模式)
            ├── entities/     # 实体页(政策/人物/活动)
            ├── graph.json    # 实体-关系图(多跳检索用)
            └── index.md      # concept 导航

# 跨校共性实体(同一政策/学区/招生新规跨多校复现)落总库,避免每校重复:
# ~/.ai-vault/wechat/_shared/entities/  + graph.json(可选,Phase 4 再定)

~/.claude/
├── skills/wechat-article-to-markdown/SKILL.md   # 转换 skill
└── commands/wx-brief.md                          # 简报生成命令(消费双轨检索)
```

> **为什么 raw/ 与 wiki/ 分开**:对齐 obwiki 的 raw-first + 渐进式披露——`raw/` 是 L2 全文层(RAG 语料、人工取证),`wiki/concepts/` 是 L1 摘要层,`INDEX.md`/frontmatter 是 L0 索引层。逻辑检索先扫 L0/L1(wiki 轨),不够再下钻 L2(raw / RAG 轨)。

> **为什么物理分库(每校一目录)**(2026-06 目标升级:北京中小学,先海淀):
> - **每校一个独立内容库**是业务北极星——校与校之间内容正交(各校自有招生/活动/通知),物理隔离便于单校查阅、单校增量、单校交付,也对齐"每校建一个公号内容库"的原始诉求。
> - **去重账本仍全局**:`state/seen_hashes.json` 跨校共享 content-hash,同一篇被多校转载只入一次(落首见的那校)。
> - **跨校检索的取舍**:分库后"逻辑题"可能要遍历多个校库的 L0/wiki。`retrieve.py` 加 `--school` 限定单校(快),不带则扫总库 `INDEX.md` 聚合各校(慢但全)。跨校共性实体(同一政策/学区招生新规)沉到 `_shared/`,避免每校重复蒸馏。
> - **命名**:子库目录 `<区>-<学校名>`(kebab 不强制,保留中文);`account` 仍从文章头部 `> 公众号:` 自动解析(公号名 ≠ 学校名,故子库按**学校**命名,公号名进 `meta.yaml` 与 frontmatter)。

---

## 3. 落地步骤(Claude Code 分阶段执行 + 验收锚点)

### Phase 1 · 安装转换引擎
**目标**:`to-markdown` 可用,Camoufox 浏览器一次性下载完成。

```bash
# CLI 安装(Mac mini M4)
uv tool install wechat-article-to-markdown

# 同时装为 Claude Code skill(二选一,推荐手动法更可控)
mkdir -p ~/.claude/skills/wechat-article-to-markdown
curl -o ~/.claude/skills/wechat-article-to-markdown/SKILL.md \
  https://raw.githubusercontent.com/jackwener/wechat-article-to-markdown/main/SKILL.md

# 冒烟测试:首次运行会下载 Camoufox 内核(实测 298MB,非"百 MB 级")
wechat-article-to-markdown "https://mp.weixin.qq.com/s/<任一公开文章>"
```

> **实测补充(2026-06-17 真机已跑通)**:
> - Camoufox 内核体积 **298MB**(`camoufox-135.0.1-beta.24-mac.arm64.zip`),直连 GitHub release 很慢且易中断。**国内用镜像**:`curl -L -C - -o /tmp/cf.zip https://gh-proxy.com/<github-release-url>`,再用 `camoufox.pkgman.CamoufoxFetcher().extract_zip()` 解压到 `~/Library/Caches/camoufox`(或直接 `camoufox fetch` 重试)。`gh-proxy.com` / `ghfast.top` 实测可用。
> - 内核就绪后 `camoufox version` 应显示 `Up to date!`。

**验收锚点**:`<output>/<标题>/<标题>.md` 生成,图片落入 `images/` 且 MD 内为相对路径,MD 头部含「公众号 / 发布时间 / 原文链接」元信息。✅ 已真机验证(北京第二实验小学一篇,32/33 图片本地化,元信息齐全)。

**已知坑(实测修正,写进 README)**:
1. **⚠ 输出落点不是 CWD,而是包安装目录**(实测 v0.1.0 推翻了"落 CWD/output" 的预期):产物固定落在
   `~/.local/share/uv/tools/wechat-article-to-markdown/lib/python3.11/site-packages/output/<标题>/`。
   `cd` 到 `.raw/` **无效**。因此 `fetch.py` 必须:
   - 用 `python -c "import wechat_article_to_markdown as m; print(m.__file__)"` 或固定推导出包目录,定位 `output/`;
   - 抓完把产物**搬**回 `~/projects/wx-brief/.raw/output/` 再入库,并**每轮清空包目录 output/**(否则跨轮残留会污染"最新产物"判断)。
   - 更稳的做法:抓取前后 diff 包目录 `output/` 的子目录集合,新出现的那个就是本次产物(避免同名标题歧义)。
2. **首次抓取会下载 uBlock-Origin 扩展**(`Downloading addon (UBO)`),首篇即触发、非致命;偶发第二篇报 UBO SSL 失败也是**非致命警告**,文章照常保存。orchestrator 对 CLI 退出码 + 实际产物**双重判断**,不要只看 stderr/退出码。
3. **图片偶发个别失败**(实测 32/33,一张因 URL 缺协议头失败)——属正常,按"产物存在即成功"判定,失败图记日志不阻断入库。

### Phase 2 · 采集清单(按学校组织,物理分库)
**目标**:`sources.yaml` 作为唯一人工维护入口,**按学校分组**——每所学校对应一个独立子库 `schools/<区>-<学校名>/`。URL 由 exporter 导出后回填(见 §6),或人工随手收集。

```yaml
# sources.yaml — 北京中小学公号采集清单(按学校分库)
# 每个 school 落成 ~/.ai-vault/wechat/schools/<区>-<name>/ 一个独立内容库
schools:
  - name: "中关村第一小学"      # 学校名 → 子库目录名(配合 district)
    district: "海淀"            # 区,拼进子库目录:海淀-中关村第一小学
    stage: "小学"               # 学段:小学/初中/高中/完中(进 meta.yaml,便于筛选)
    account: "中关村一小"        # 公号显示名(兜底;实际 account 抓取时从文章头部自动解析)
    biz: ""                     # 可选,exporter 回填
    articles:
      - "https://mp.weixin.qq.com/s/AAAA"
      - "https://mp.weixin.qq.com/s/BBBB"
  - name: "北京市第二十中学"
    district: "海淀"
    stage: "完中"
    account: "北京市第二十中学"
    articles:
      - "https://mp.weixin.qq.com/s/CCCC"
```

- **分库键** = `<district>-<name>`;同名学校靠 district 区分。
- `account` 不必填准,抓取时以文章头部 `> 公众号:` 为准覆盖(公号名常 ≠ 校名)。
- `stage`/`district` 落 `meta.yaml`,供后续按区/学段批量跑与检索过滤。

**验收锚点**:`fetch.py --dry-run` 能解析清单、按学校分组打印各校待抓 URL 数与全局去重后净增数;`--school "海淀-中关村第一小学"` 能只跑单校。

### Phase 3 · 抓取 + 入库 orchestrator(落 raw)
**目标**:`fetch.py` 串起 抓取 → frontmatter → 去重 → 落 `schools/<区>-<校>/raw/` → 更新本校 INDEX + 总 INDEX(L0 索引层)。

> 这一层只负责把原文不可变地落进本校 `raw/`,**不做蒸馏、不灌 RAG**——那是 Phase 4 / Phase 5 两条沉淀轨的事。raw-first。

接口契约(命令行):
```
python fetch.py            # 跑全量 sources.yaml,去重后只处理净增
python fetch.py --school "海淀-中关村第一小学"
python fetch.py --dry-run  # 只解析+去重统计,不实抓
python fetch.py --limit 6 --sleep-min 8 --sleep-max 20  # 安全小批量扩抓
```

orchestrator 骨架(Claude Code 据此补全):
```python
# fetch.py — 抓取层 + 入库层(物理分库,只落 raw)
# 核心职责:调用 to-markdown,加 frontmatter,content-hash 去重,原子落 schools/<区>-<校>/raw/
import hashlib, subprocess, shutil, json, yaml
from pathlib import Path
from datetime import date

PROJECT = Path.home() / "projects/wx-brief"
RAW = PROJECT / ".raw"
VAULT = Path.home() / ".ai-vault/wechat"
SCHOOLS_DIR = VAULT / "schools"     # 每校一个独立子库
LEDGER = PROJECT / "state/seen_hashes.json"

def school_dir(key: str) -> Path:
    return SCHOOLS_DIR / make_slug(key)

def _pkg_output_dir() -> Path:
    # ⚠ 实测:v0.1.0 把产物固定写到「包安装目录」的 output/,不是 CWD。cd 无效。
    import wechat_article_to_markdown as m
    return Path(m.__file__).resolve().parent / "output"

def fetch_one(url: str) -> Path:
    RAW.mkdir(parents=True, exist_ok=True)
    pkg_out = _pkg_output_dir()
    before = set(pkg_out.glob("*/")) if pkg_out.exists() else set()
    subprocess.run(
        ["wechat-article-to-markdown", url],
        check=False,  # check=False:退出码不可靠,靠产物判断;cwd 不影响落点
    )
    # diff 包目录 output/ 新增的子目录 = 本次产物(避免同名标题歧义)
    after = set(pkg_out.glob("*/"))
    new_dirs = after - before
    art_dir = new_dirs.pop() if new_dirs else max(after, key=lambda p: p.stat().st_mtime)
    produced = next(art_dir.glob("*.md"), None)
    assert produced, f"抓取无产物: {url}"  # 实际产物是成功判据,而非退出码
    # 搬回项目 .raw/output/ 统一管理,并清掉包目录该产物(每轮不留残留)
    dest = RAW / "output" / art_dir.name
    dest.parent.mkdir(parents=True, exist_ok=True)
    if dest.exists():
        shutil.rmtree(dest)
    shutil.move(str(art_dir), str(dest))
    return next(dest.glob("*.md"))

def content_hash(md_text: str) -> str:
    # 去除 frontmatter/图片路径噪声后哈希,与 WeChat 监控系统的去重口径保持一致
    return "sha256:" + hashlib.sha256(md_text.encode("utf-8")).hexdigest()

def to_raw(raw_md: Path, school: dict, key: str, url: str, ledger: dict) -> bool:
    body = raw_md.read_text(encoding="utf-8")
    h = content_hash(body)
    if h in ledger:            # 内容级去重:同文重发/转载只入一次
        return False
    # 入库四操作判定(借 obwiki 第 4 铁律):同 source_url 已存在 → 视为重发/修订
    #   走 UPDATE(新文件 supersedes 旧文件);全新 → ADD;库里等价 → 上面已 NOOP。
    #   DELETE 永不自动,冲突只标记 needs_review,不消解(人在环)。
    fm = build_frontmatter(body, school, url, h)   # 见 §4 schema,含 distilled/rag_indexed 状态位
    slug = make_slug(raw_md.stem)
    raw_store = school_dir(key) / "raw"
    out = raw_store / f"{fm['publish_date']}__{make_slug(fm['account'])}__{slug}.md"
    out.parent.mkdir(parents=True, exist_ok=True)
    out.write_text(fm_dump(fm) + "\n" + body, encoding="utf-8")
    # 图片同步搬运并改写相对路径(images/ 随文件走)
    move_images(raw_md.parent / "images", out.parent / f"{out.stem}.assets")
    ledger[h] = {"url": url, "school": key, "file": out.name}
    return True

# main: load sources.yaml → for each url: fetch_one → to_raw →
#       清空 .raw → 写 ledger → 重建本校 INDEX + 总 INDEX(L0:按学校/date 分组)
```

**验收锚点**:
- 重复跑两次,第二次净增 = 0(去重生效);
- `schools/<区>-<校>/raw/` 内每篇有合法 frontmatter(含 `distilled: false` / `rag_indexed: false`),图片在同名 `.assets/` 内,MD 渲染无坏链;
- 本校 `INDEX.md` 按日期列 raw 概览,总 `INDEX.md` 按学校列篇数;表格内使用标准 Markdown 相对链接(避免 wikilink alias 的 `|` 打断表格),wiki/entity 页内再使用 Obsidian wikilink;
- 每校自动创建 `wiki/index.md` 与 `wiki/entities/<学校名>.md` 骨架;raw 不硬插双链,双链密度由 Phase 4 的 concept/entity 层承担。
- 批量扩抓必须限速串行:`--limit` 小批量、篇间随机 sleep、每 N 篇 cooldown;失败写 `state/failures.json`,避免访问限制时盲目重试。

### Phase 4 · 沉淀轨 A · 蒸馏成 wiki(distill.py)
**目标**:`distill.py` 把 `raw/` 里 `distilled: false` 的文章蒸馏成 `wiki/concepts/` 的 concept 页,并建链。**独立步骤、可批量**,与简报解耦。

> 这是 obwiki 理念注入的核心一步:**逻辑检索这一轨的语料就是这里产出的 concept 页 + graph**。学校公号互链天然密集(同校多文、政策连续、人物/活动复现),正好吃链接密度红利。

接口契约:
```
python distill.py             # 蒸馏所有 distilled:false 的 raw
python distill.py --school "海淀-北京市海淀区第二实验小学"
python distill.py --since 2026-04-01
```

蒸馏协议(借 obwiki distill,Claude Code 据此实现 prompt + 落地逻辑):
1. **判定四操作**:对每篇 raw,先扫 `wiki/concepts/` 的 L0(标题 + summary)找相关页,判定:
   - `ADD` 全新主题 → 新建 concept 页;
   - `UPDATE` 已有页缺这条 → 融进去,`updated:` 改今天;
   - `MERGE` 多页相关 → 分别补并互链;
   - `NOOP` 已有等价信息 → 不动(只把 raw 的 `distilled` 翻 true)。
2. **concept 页格式**(karpathy 模式,200-400 字):一句话定义 + 结构化要点 + `[[链接]]`(每条带"为什么相关",禁裸链)+ 来源 `[[raw 文件]]` 回引。
3. **抽实体/关系**写 `wiki/graph.json`(学校 / 政策 / 人物 / 活动作为实体,供多跳检索)。
4. 模型走 LiteLLM alias `local-qwen3-14b`(蒸馏量大、本地便宜);抽实体可同模型。
5. 完成后 raw frontmatter `distilled: false → true`。

> 批量积压(≥ 8 篇 raw 待蒸)时,按 obwiki 的 [`batch-processing`](../../myskills/obwiki/references/operations/batch-processing.md):主 agent 持文件清单 + 决策,子 agent 分批读原文回传 concept,避免主上下文爆炸。

**验收锚点**:
- 每篇 raw 蒸出/更新对应 concept 页,`wiki/index.md` 可导航;
- 链接密度可量(平均每页 ≥ 3 条带理由的 `[[链接]]`),无孤立页;
- 重跑幂等:`distilled: true` 的不重复处理。

> **实测补充(2026-06-21)**:Phase 4 首版已落地到 `~/projects/wx-brief/distill.py`。当前采用确定性规则版(标题/正文关键词归主题 + 原文句子摘要),已把 10 校 / 42 篇 raw 全量蒸馏为 29 个 `wiki/concepts/*.md` 与 10 个 `wiki/graph.json`;raw frontmatter `distilled=true` 且 `concepts/entities` 已回填。重跑 `distill.py --dry-run` 显示 0 待处理;后续接入 LiteLLM `local-qwen3-14b` 时替换摘要/分类函数,不改幂等协议与文件布局。

### Phase 5 · 沉淀轨 B · 灌 LightRAG(index_rag.py)
**目标**:`index_rag.py` 把 `raw/` 里 `rag_indexed: false` 的原文切片灌进现有 LightRAG 向量库,作为**精准/取证检索**的语料源。

> 与轨 A 互补:wiki concept 是"提炼后的逻辑",RAG 是"原文现场"。要原话、具体数字、政策细节、谁在哪天说了什么 → 走这轨。

接口契约:
```
python index_rag.py           # 灌所有 rag_indexed:false 的 raw
python index_rag.py --reindex <file>   # 单篇重灌(修订后)
```

要点:
- 复用现有 LightRAG 实例(沿用其 embedding/切片配置,不另起一套);文档 id = content_hash,保证幂等、可重灌。
- 切片携带 metadata(`account` / `publish_date` / `source_url` / `raw 文件名`),让 RAG 召回能回引原文与 wiki。
- 灌完 raw frontmatter `rag_indexed: false → true`,并记 `state/rag_indexed.json`。

**验收锚点**:
- 给一个原文细节查询(如"某校 X 月运动会具体安排"),LightRAG 能召回正确原文片段且带 metadata 回引;
- 重跑幂等:同 content_hash 不重复灌;修订篇 `--reindex` 能覆盖旧切片。

> **实测补充(2026-06-21)**:`~/projects/wx-brief/index_rag.py` 首版已实现切片、metadata、`doc_id=content_hash` 幂等、`--school/--since/--limit/--reindex` 与两种后端。默认 `staging` 后端只写 `state/rag_staging.jsonl` + `state/rag_staging_state.json`,用于验证切片,**不会**翻 raw `rag_indexed`；真实 `lightrag-api` 后端需显式传 `--endpoint`,并等待 LightRAG 处理完成后才写 `state/rag_indexed.json`、翻 `rag_indexed=true`。本机已安装 `lightrag-hku` 到 `~/projects/wx-brief/.venv`,服务脚本为 `bin/lightrag_server_local.sh`;embedding 使用 `nomic-embed-text:latest` 且 `EMBEDDING_DIM=768`。真实文章入库当前卡在本地 `qwen3:14b` entity extraction 超时,所以 Phase 5 处于“脚本/staging/安装已完成,真实灌库待更快 LLM 或远端 LLM 验证”状态。LightRAG/Ollama 进程已停止,端口 `9621`/`11434` 不应常驻。

### Phase 6 · 检索分流(retrieve.py)
**目标**:`retrieve.py` 是双轨检索的统一入口,按查询类型分流——**逻辑/概念问题先走 wiki,精准/取证问题走 LightRAG**——供 `/wx-brief` 和人工查询共用。

接口契约:
```
python retrieve.py "XX中学今年的德育主线是什么"        # 逻辑题 → wiki 轨
python retrieve.py --raw "X月X日运动会的具体项目安排"   # 精准题 → 强制 RAG 轨
python retrieve.py --explain "..."                     # 打印走了哪轨 + 命中页/片段,便于调参
```

分流协议(借 obwiki 第 6/9 铁律:分层检索 + 渐进式披露):

| 查询类型 | 走哪轨 | 手段 |
| :--- | :--- | :--- |
| **逻辑 / 概念 / 归纳 / 多跳** | **wiki 轨(默认)** | L0 扫 frontmatter/index → L1 读候选 concept 页 → 反思缺口 → 多跳走 `graph.json` 遍历 → 仍缺证据才下钻 L2(raw / 转 RAG) |
| **精准 / 取证 / 原话 / 具体数字** | **RAG 轨** | LightRAG 向量召回原文片段 + metadata 回引,直接给现场证据 |

- **默认先走 wiki**:逻辑题在 concept 页 + graph 上就能答,快且省 token;wiki 答不全(信息缺口门控触发)再转 RAG 取原文。
- **`--raw` 强制 RAG**:明确要原话/细节时跳过 wiki。
- **查询即回写**(obwiki 第 5 铁律):综合出 wiki 里没有的新认识,默认补回最相关 concept 页(`updated:` 改今天)并告知补在哪——查询让库越查越厚。

**验收锚点**:
- 一个逻辑题默认走 wiki 轨并在 concept 层答出(`--explain` 显示未触 RAG);
- 一个原文细节题走 RAG 轨召回正确片段;
- wiki 答不全的题能自动降级到 RAG 补证。

### Phase 7 · 简报命令(消费双轨检索)
**目标**:`/wx-brief` 读取区间内文章,**优先消费 wiki concept 做主题聚类与逻辑骨架,按需向 RAG 取原文佐证**,产出简报。

`~/.claude/commands/wx-brief.md` 要点:
```
入参:--since <date> | --school <区-学校名> | --range <d1..d2>
流程:
  1. L0 扫 INDEX.md,筛 publish_date 命中区间且 status: raw 的文章
  2. 逐篇摘要 → LiteLLM alias `local-qwen3-14b`(本地、便宜、批量);
     已蒸馏的优先复用 wiki concept 页,避免重复摘要
  3. 跨篇综合(主题聚类 + 重点提炼)→ 走 retrieve.py 的 wiki 轨拿 concept + graph
     做逻辑骨架 → LiteLLM alias `claude`(质量优先)
  4. 需要原话/具体数字佐证时,按需走 retrieve.py --raw(RAG 轨)取原文片段嵌入简报
  5. 输出 ~/.ai-vault/briefs/<range>__wechat-brief.md(每个论断带 [[concept]] / 原文链接回引)
  6. 被纳入简报的文章 frontmatter.status: raw → briefed(避免重复进简报)
约束:所有模型调用走 models.yaml 别名,不硬编码 endpoint
```

**验收锚点**:给定日期区间,产出结构化简报(按学校/主题分节,论断带 `[[concept]]` 与原文链接回引),对应文章 status 翻转为 `briefed`,且简报里既有 wiki 逻辑骨架、也有 RAG 原文佐证。

---

## 4. Vault frontmatter schema(与 `~/.ai-vault/` 对齐)

```yaml
---
type: wechat-article
account: "XX中学"
account_biz: ""              # 可选
title: "原文标题"
author: "署名作者"
publish_date: 2026-04-10     # 从正文元信息解析
fetch_date: 2026-06-17
source_url: "https://mp.weixin.qq.com/s/AAAA"
content_hash: "sha256:..."
tags: [公号, 学校简报]
status: raw                  # raw → briefed,raw-first 原则(简报消费状态)
distilled: false             # 沉淀轨 A:是否已蒸馏成 wiki concept 页
rag_indexed: false           # 沉淀轨 B:是否已灌入 LightRAG
concepts: []                 # 蒸馏后回填:本文贡献/更新的 concept 页 [[链接]]
entities: []                 # 抽取的实体(学校/政策/人物/活动),供图谱与 L0 检索
supersedes: null             # 若为修订/重发,指向被替代文件
needs_review: false          # 与已有页冲突时标 true,人在环消解(不自动)
---
```

> **三个状态位对应两条沉淀轨 + 一个消费**:`distilled`(轨 A·wiki)、`rag_indexed`(轨 B·RAG)、`status`(简报消费)。三者独立翻转,各自幂等——这正是 obwiki raw-first + 分轨处理的落地。L0 检索只读 frontmatter(`title`/`summary`/`entities`/`updated`),不碰正文。

---

## 5. 合规与稳健性(低风险,但写清楚)

- **内容性质**:抓取的是公开发布的公号文章,自用于咨询简报/知识库,低频。`to-markdown` 仅请求公开文章页,不解密、不触客户端数据库——与你 WeChat 监控选 Accessibility API 而非 DB 解密的合规取向一致。
- **频率**:Camoufox 已抗指纹,但仍按低频跑(批量间隔随机化,不要并发轰)。orchestrator 在 URL 间加 2–5s 随机 sleep。
- **失败可重入(三轨各自幂等)**:ledger + 产物双判据使抓取幂等;`distilled` / `rag_indexed` 状态位使蒸馏轨、RAG 轨各自幂等。任一轨中断后重跑只补未完成的,不重复处理。
- **人在环**(obwiki 第 8 铁律):DELETE 永不自动;蒸馏时发现 concept 页冲突(两篇说法矛盾)标 `needs_review`,出清单给人确认,不自动消解。

---

## 6. 发现层 · exporter(wx-exporter)

定位分两种:
- **新文章发现依赖**:如果学校集合不变、但要持续抓取新发布内容,需要 `wx-exporter` 或等价发现层拿到新 URL;`fetch.py` 只消费 `sources.yaml` 已知 URL,不能自行发现公号新文章。
- **可选增强**:只有做阅读量/在看/评论 KPI 简报,或全量回填某公号历史文章时,才需要 exporter 的 KPI/全量能力。

部署(挂你现有 Dokploy + Traefik):
```bash
docker pull ghcr.io/wechat-article/wechat-article-exporter:latest
# 经 Dokploy 起服务,Traefik 反代到内网域名
```
本机现状:Docker 容器 `wx-exporter` 已运行,地址 `http://localhost:3000/dashboard/account`,镜像 `ghcr.io/wechat-article/wechat-article-exporter:latest`,端口 `3000`。用法:浏览器打开 → **用自己的微信公众号扫码登录**(这是硬门槛,需有一个注册公号,个人订阅号即可)→ 搜目标公号 → 导出。

与主链路衔接的两种用法:
1. **只取 URL 清单**:用 exporter 列出某公号全部文章 → 导出 URL → 回填进 `sources.yaml` → 仍走 to-markdown 主链路转 MD(保持 vault 入库口径统一)。
2. **取 KPI 数据**:导 Excel(阅读量/评论)→ 单独喂给 KPI 简报,不进知识 vault。

> 当前项目如果只抓已知 URL,可以不经过 exporter;如果要“学校不变,自动补新发文章”,则 exporter/等价发现层是事实依赖。主链路仍保持 raw-first:exporter 只产 URL/KPI,正文统一由 `fetch.py` + to-markdown 入库。

---

## 7. 总验收 Checklist

- [ ] `to-markdown` CLI + Claude Code skill 安装,冒烟测试过单篇
- [ ] `sources.yaml` 可解析,`--dry-run` 统计正确
- [ ] `fetch.py` 幂等:二次运行净增 = 0,raw 落 `raw/` 且 frontmatter 含三状态位
- [ ] **轨 A** `distill.py`:raw → concept 页,链接密度达标(每页 ≥ 3 条带理由 `[[链接]]`),`distilled` 幂等
- [ ] **轨 B** `index_rag.py`:raw 灌入现有 LightRAG,原文细节可召回且带 metadata,`rag_indexed` 幂等
- [ ] **检索分流** `retrieve.py`:逻辑题默认走 wiki 轨、精准题走 RAG 轨,`--explain` 显示走向正确
- [ ] `/wx-brief` 消费双轨(wiki 逻辑骨架 + RAG 原文佐证)产出简报,论断带回引,status 正确翻转
- [ ] README 记录两个已知坑(输出目录 / ublock 非致命警告)
- [ ] (可选)exporter 模块部署文档就位,但默认不启用

---

## 附:升级路径(暂不实现,留备忘)

- **持续监控**:若日后需要"新文章自动入库",可引入 RSS 桥(wechat2rss / werss 类)替代人工维护 `sources.yaml` 的发现环节,产出 URL → 喂 `fetch.py`。低频场景暂不需要。
- **wiki 轨深化**:concept 页稳定后,定期跑 obwiki 的 `evolve`(跨库蒸馏、分类/结构演化、补链体检),把 wechat vault 并入更大的个人知识库治理。
- **RAG 轨升级 GraphRAG**:LightRAG 已是 graph-augmented;若日后跨语料主题归纳不够,再上 obwiki 指路的重型 `graphify`(社区检测/HTML)。按 obwiki 第 6 铁律,轻手段够用就别上重的。
- **检索分流自动化**:`retrieve.py` 的"逻辑 vs 精准"分流目前靠 `--raw` 显式或简单启发式;后续可用一个轻分类(LiteLLM 小模型)自动判定查询类型再路由。

---

## 附:与 obwiki 的关系(借鉴边界)

本方案**借鉴 obwiki 的构建原则**(渐进式披露、链接密度、四操作判定、分层检索、查询即回写、人在环),但**不直接依赖 obwiki skill 运行**——wechat vault 自带 `fetch/distill/index_rag/retrieve` 四脚本闭环,可独立跑。

- **可选融合**:若把 `~/.ai-vault/wechat/` 直接当作一个 obwiki vault(它的 `raw/` + `wiki/` 布局已对齐 obwiki),则 `/obwiki ingest/retrieve/evolve` 能直接接管,wx-brief 的脚本退化为"专用快捷入口"。
- **不重复造**:重型图谱、跨库蒸馏、分类演化这些 obwiki 已有的能力,本方案只指路不重实现。

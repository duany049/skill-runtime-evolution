# ΩmegaWiki — Runtime Contract

编辑 `i18n/zh/CLAUDE.md`,不要改根目录下的副本。运行 `./setup.sh --lang zh` 同步。

## 仓库布局

- `wiki/` — 产物面。`index.md` 是目录;`log.md` 是 append-only;每类实体一个子目录;`wiki/graph/` 自动生成。
- `runtime/` — 契约源(schema + policy + templates)。修改任何规则前先读 `runtime/CLAUDE.md`。
- `raw/` — 用户自有 `{papers,notes,web}/`(只读)+ skill 可写的 `discovered/`、`tmp/`。
- `tools/` — Python 助手(`research_wiki.py` 是 wiki 引擎,`lint.py` 是校验器)。

完整目录树:`docs/runtime-directory-structure.zh.md`。

## 链接语法

Wikilink:`[[slug]]`。slug 全小写、连字符分隔、无空格。

## 硬规则

1. `raw/{papers,notes,web}` 归用户所有,只读。skill 只能向 `raw/discovered/` 或 `raw/tmp/` 追加。
2. `wiki/graph/` 是派生态。仅通过 `tools/research_wiki.py`(`add-edge`、`add-citation`、`rebuild-*`)修改。
3. `wiki/log.md` 是 append-only。绝不就地重写。
4. 写正向链接 → 同步写反向链接。完整规则在 `runtime/schema/xref.yaml`。
5. 用户面 skill 参数(skill `argument-hint` 里列出的 flag)归用户所有。不得仅根据仓库状态擅自补出、翻转或删除它们。用户未提供时,只有 skill 文档化了省略行为才用默认值;否则询问用户。

## 查阅索引

| 需要 | 去哪 |
|---|---|
| 页面 frontmatter 字段、enum、默认值、生命周期 | `runtime/schema/entities.yaml` |
| 页面正文章节结构                                | `runtime/templates/{kind}.md.tmpl` |
| 边类型、属性、方向、confidence                | `runtime/schema/edges.yaml` |
| 正向 → 反向链接规则                            | `runtime/schema/xref.yaml` |
| slug 规则、ownership、edge 存储位置            | `runtime/schema/conventions.yaml` |
| 各 skill 对字段/边的写权限                     | `runtime/policy/writers.yaml` |
| 改契约本身 / 重新 regen                        | `runtime/CLAUDE.md` |

## Python 环境

按优先级:`.venv/bin/python`(Windows 上 `.venv/Scripts/python.exe`)→ 当前激活的 conda 环境 → `python3`(Windows 上 `python`)。tools/ 通过 `tools/_env.py` 自动从 `~/.env` 和项目根 `.env` 加载 API key。

---

## 语言策略（KnowledgeBase 强制）

无论用户用什么语言提问、原始论文用什么语言、上游 skill prompt 用什么语言，
**所有你生成的内容都必须用中文**：

- 给用户的回复：中文
- Wiki 页面正文（papers / concepts / methods / topics / ideas / experiments / Summary / foundations）：中文
- 生成的 LaTeX 论文、poster、survey：中文，除非用户在 runtime 明确要求英文
- 终端日志、状态消息、报告：中文
- 代码注释和说明：中文

**保留英文原文的例外**（不强行翻译）：

- 学术术语：LoRA、attention、transformer、reinforcement learning、in-context learning、prompt、token、embedding 等
- 论文标题、作者姓名、机构名、会议名（NeurIPS / ICML / ACL ...）、URL、文献引用
- 代码标识符：变量名、函数名、类名、模块名
- 文件路径、命令行参数、环境变量、配置 key
- arXiv ID、DOI、commit hash 等技术标识符

写法：中文叙述 + 英文术语原样嵌入，例如 "本文提出基于 attention 的 LoRA 微调方法" 而不是 "本文提出基于注意力机制的低秩适配方法"。

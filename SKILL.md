---
name: chinese-citation-verifier
description: >-
  中文文献的元数据核验、补全与摘要相关度检查（与 academic-integrity-gate 互为姊妹篇，
  英文医学走 pubmed MCP，中文走本 skill）。从 OpenAlex × Crossref 双源交叉验证取规范元数据，
  ChinaXiv 兜底预印本；按字段级仲裁规则打标「已核验·双源 / 已核验·单源 / 冲突 / 未核验」；
  可选声明级核查（该文献摘要是否真支持论点）；查不到绝不补全；信源失联即停。
  显式触发：核验中文文献 / 查证中文引用 / 中文文献核查 / 补全中文元数据 /
  这些中文文献是真的吗 / verify Chinese citations / 中文 citation audit。
  自动嗅探（保守）：用户粘贴 3+ 条中文参考文献且问核查类问题；AI 即将输出中文参考文献段落。
  不适用：英文文献（→ academic-integrity-gate）；纯中文随笔创意。
  v2 起接入本地 Zotero MCP（mcp__zotero__*），命中且元数据完整则跳过网络层；
  未配置 Zotero MCP 或库内未命中时退化到 v1 网络层（OpenAlex × Crossref + ChinaXiv）。
---

# chinese-citation-verifier — 中文文献核验与元数据补全

> 一句话：中文文献的「真相源 + 双源仲裁 + 显性标注 + 信源失联即停」。
> 与 `academic-integrity-gate` 同源同骨——同一套诚信门，不同的真相源。
> 设计基于真实中文学术核验场景的需求抽象（特别是 TCM / 中医正骨等 CNKI 独家覆盖的领域）。

## 何时启用 / 何时不启用

**启用：**
- 用户显式触发（见 description 中触发短语）。
- AI 即将输出**中文参考文献段落 / 中文文献清单 / 含中文引用的稿件**。
- 用户粘贴 3 条以上中文文献候选 + 问"对不对 / 查一下 / 补全 / 相关度"类问题。
- 与 `academic-integrity-gate` 串联：英文文献走 AIG + pubmed，中文文献走本 skill。

**不启用：**
- 英文文献（走 academic-integrity-gate）。
- 中文非学术随笔、创意写作。
- 用户已声明"经 Zotero+茉莉花 富集且无需复核"的单条核验。
- 中文偶有一次性提及某文献名（非核查 / 非出稿）。

## 依赖与降级

| 组件 | 类型 | 缺失时行为 |
|---|---|---|
| `mcp__zotero__*` 工具族 | 用户全局 Claude Code MCP（非本 skill 自带） | 跳过 Step 0，走 v1 网络层；报告顶部加配置提示行 |
| Zotero 客户端 + 茉莉花插件 | 用户机器侧 Zotero 客户端 | MCP 通但库内中文条目元数据空壳 → B 态「待补全」 |
| OpenAlex / Crossref / ChinaXiv | 公开 API | 任一失联：按既有「信源失联即停」规则处理 |

配置自检与降级路径详见 `SETUP_Zotero.md`。

## 覆盖局限提醒（**首次启用时主动告知用户**）

OpenAlex × Crossref 双源对中文文献的覆盖**远非均匀**。基于 20260606 smoke test 实测（3 条中医正骨实战引用 0/3 命中）+ 既有领域知识，分领域估算：

| 领域 | 估计覆盖 | 主因 |
|---|---|---|
| 西方医学中文期刊（中华××杂志、Science China 系列） | 70–80% | 多有国际 DOI |
| STEM / 工科中文期刊 | 50–70% | 取决于出版方 |
| **中医药 / 中医正骨 / TCM** | **<10%** | **大量 CNKI 独家、未注册国际 DOI** |
| 社科 / 人文中文期刊 | <20% | 少 DOI |
| 学位论文 / 会议 / 灰文献 | <5% | 基本不覆盖 |

**强制行为**：
1. **首次启用本 skill 时**，先抽样判断用户文献所属领域。若以**中医药 / TCM / 国内队列 / 学位论文**为主，**先打断并主动告知**："本领域 v1 命中率会很低，建议优先用 Zotero 本地层（v2，规划中）或 CNKI / SinoMed 人工核验。仍要跑就跑，但 0/N 命中是正常的，不代表文献不存在。"
2. **禁止**默默跑、默默返回"全部未核验"——这会让用户误以为"国际索引查不到 = 文献不存在 / 是假引用"。
3. 若用户研究主域为 **骨科 / 中医正骨 / 国内队列 / TCM**，默认按"高几率落空"处理，**v2 起优先走 Zotero 本地层**（茉莉花对接 CNKI/万方/维普可显著提升 TCM 域命中率）；Zotero 也未命中再退到 v1 网络层。

注："DOI 风格内部编号"陷阱：prefix 形如 `10.19891`、`10.13862` 等可能是国内出版社内部编号，**未在 Crossref 注册**，调用时会 HTTP 404。这不是网络错误，是"不是真的 DOI"。

**v2 实测对比（2026-06-06，3 条 TCM 引用）**：v1 网络层 0/3 命中 → v2 加入 Zotero 层后 1/3 完整命中（A 态）+ 1/3 空壳待补全（B 态）+ 1/3 仍未命中。Zotero + 茉莉花是 TCM 域真正的可靠信源。

## 五条铁律（继承 academic-integrity-gate，本地化）

1. **唯一真相源** — 中文文献是否存在、字段是否正确，**只承认网络真相源查证结果**。模型「记忆 / 训练知识」**不构成证据**。本 skill v1 的真相源链：
   - 主：**OpenAlex API** + **Crossref API**（双源并行）
   - 兜底：**ChinaXiv**（仅预印本场景）
   - v2 接入：本地 Zotero（茉莉花富集后回读）

2. **双源交叉验证** — 元数据字段必须在**至少两个独立信源**取得一致后才算「已核验·双源」；只一源命中 = 已核验·单源（需明确标源）；字段冲突 = 冲突报告，**不擅断**。

3. **查不到不补** — 任何字段（标题 / 作者 / 期刊 / 年 / 卷 / 期 / 页 / DOI / 摘要）真相源未给，**绝不**凭记忆或推测补全。打标「未核验·信源失联」。

4. **显性标注** — 每条引用必须出现下列标签之一：
   - `[已核验·双源]` — OpenAlex + Crossref 字段一致
   - `[已核验·单源·OA]` / `[已核验·单源·CR]` / `[已核验·单源·ChinaXiv]` — 仅一源命中且自洽
   - `[冲突·{字段名}]` — 多源命中但字段不一致，列两份原文给人决
   - `[未核验·信源失联]` — 真相源全部未命中或不可用

5. **信源失联即停** — 任一次真相源调用失败 / 超时 / 全空响应，停下报告，不得用记忆顶替继续输出。两源同时不可用 = 硬停。

## 工作流（Step 0 Zotero 优先 + Step 1-6 网络层兜底）

### Step 0：Zotero 本地层（v2 起）

**前置**：检测 `mcp__zotero__*` 工具族是否可用（调用 `zotero_list_libraries` 一次即可，错误 → 跳过本步）。

**查询**：
- 首选 `zotero_search_items(query=<短关键词>, qmode="everything", limit=5)`；命中 → `zotero_get_item_metadata(item_key, format="json")`
- 未命中 → `zotero_advanced_search` 二次尝试（title contains + itemType is journalArticle）
- 仍未命中 → 状态 C·未命中，**沉默通过**本层，进入 Step 1

**三态分流**（详见 `RULES_zotero_layer.md`）：

| 状态 | 触发 | 处置 |
|---|---|---|
| A·完整命中 | search 命中 + 元数据完整判据全过（title / creators ≥1 / 4 位年 / publicationTitle 全有） | 标 `[已核验·Zotero·完整]`，**跳过 Step 2-3 网络层**，直进 Step 5/6 |
| B·空壳待补全 | search 命中但元数据缺必要项 | 标 `[已存在·Zotero·待补全]`，**暂停本条**，提示用户在 Zotero 中跑茉莉花后重试；**禁止**凭 title 补字段 |
| C·未命中 | search + advanced_search 均 0 条 | 不打标，进入 Step 1-3 网络层 |

字段映射、查询启发式、失败信号详见 `REFERENCE_Zotero.md`。

**MCP 不可用**：跳过 Step 0；在最终报告顶部加一行：
> 提示：未检测到 Zotero MCP。若文献已在本地 Zotero 中，配置后命中率会显著提升（见 `SETUP_Zotero.md`）。

### Step 1：输入归一化
- 提取候选字段：标题 / 第一作者 / 期刊 / 年 / 卷 / 期 / 页 / DOI
- 中文归一化：
  - 繁体 → 简体
  - 全角 → 半角（数字、字母、标点）
  - 折叠多空格、去前后空白
  - 中文标点统一（《》、""''等保留；其他统一为半角）
- 输出：每条引用一条结构化记录

### Step 2：信源查询（并行）
- **OpenAlex**：见 `REFERENCE_OpenAlex.md`
  - 有 DOI 用 DOI 直查
  - 无 DOI 用 `search=` 标题 + `filter=author.display_name.search:` 第一作者
- **Crossref**：见 `REFERENCE_Crossref.md`
  - 有 DOI 用 `/works/{doi}` 直查
  - 无 DOI 用 `query.bibliographic=` 综合检索
- 两者**并行**调用（WebFetch 并行）

### Step 3：双源仲裁
按 `RULES_cross_validation.md` 的仲裁表打标。字段级比对：标题 / 第一作者 / 期刊 / 年 / 卷期页 / DOI。

注：Step 0 已 A 态命中时，Step 2-3 整段跳过；Step 0 B 态时本条暂搁置，**不**自动跑 Step 2-3。

### Step 4：兜底（条件触发）
若 OpenAlex + Crossref 都未命中 **且**：
- 引用文本含「预印本 / preprint / chinaxiv」字样 → 查 `REFERENCE_ChinaXiv.md`
- 否则**跳过兜底**，进入 Step 6 出未核验报告

### Step 5：摘要相关度（可选）
**仅当**用户在原任务中表明「这篇支持 / 反对 论点 X」时启动，否则跳过。
- 取摘要：OpenAlex 的 `abstract_inverted_index` 重构 → 重组中文摘要；Crossref 通常无中文摘要
- 按 `RULES_abstract_check.md` 判断：支持 / 部分支持 / 不支持 / 摘要不可得
- **必须给原文锚点**（摘要片段引用，禁止"读后感"式判断）

### Step 6：出报告
每条引用按 `examples/` 中三种模板（双源命中 / 字段冲突 / 信源失联）出报告。
**禁令**：未核验绝不混入"已核验"列表；冲突绝不擅断；不报告失联就跳过此条。

## 输出 schema（每条）

```yaml
原始引用: <用户原文>
核验状态: 已核验·Zotero·完整 | 已存在·Zotero·待补全 | 已核验·双源 | 已核验·单源·{OA|CR|ChinaXiv} | 冲突·{字段} | 未核验·信源失联
信源命中: [Zotero | OpenAlex | Crossref | ChinaXiv]   # 实际命中的
zotero_item_key: <仅 Zotero 命中时>
规范元数据:
  标题: ...
  作者: [..., ...]
  期刊: ...
  年: ...
  卷: ...
  期: ...
  页: ...
  DOI: ...
冲突字段:                                  # 仅冲突状态出现
  <字段名>:
    来源A: ...
    来源B: ...
摘要: <如可得且做了相关度检查>
摘要相关度: 支持 | 部分支持 | 不支持 | 摘要不可得 | 未做此项
摘要锚点: "<原文 1-2 句>"                   # 仅做了相关度时
```

## Hard Rules

- 核验在前，措辞 / 润色在后——顺序不可反。
- 单源命中**必须**打 `单源·{来源}` 标签，不得隐藏单源事实。
- 字段冲突时**禁止**自行选边，必须报告两源原文给人决。
- 真相源失联 = 停，不是降级到记忆。
- 中英文混合稿：本 skill 只处理中文条目，英文条目移交 `academic-integrity-gate` + pubmed MCP。
- 摘要相关度判断**必须**引用摘要原文片段做锚点，禁止"我感觉支持"式判断。
- 摘要不可得就是不可得，**不许**用标题 + 期刊推断摘要内容。
- 用户给定声明与摘要不符时，必须如实指出（不为讨好用户而修饰）。
- **Zotero 空壳条目**（B 态）**禁止**自动降级到网络层"补全"——这等同于凭 title 猜字段，违反铁律三。默认搁置等用户跑茉莉花。
- Zotero A 态完整命中**不升格为「双源」**：茉莉花是用户机器侧的二手数据汇集，技术上仍是单源；但在 TCM / 国内队列域，其可信度高于 OpenAlex+Crossref 双源，故跳过网络层是合理优化，不是放松诚信门。

## 与既有 skill / 工具的协同

| 需求 | 调用 |
|---|---|
| 英文医学文献真相源 | `academic-integrity-gate` + `pubmed` MCP |
| 通用 OpenAlex 查询 | `openalex-database`（如已装；本 skill 直接走 API 不依赖） |
| 引用格式 / Zotero 衔接 | `citation-management`（v2 集成 Zotero 后联动） |
| 统计 / 时间一致性 | `statistics-verifier` / `scientific-critical-thinking` |
| 去 AI 味（核验通过后） | `deai`（顺序：先核验，后去味） |

## v1 → v2 路线（占位）

| 版本 | 范围 |
|---|---|
| v1 | OpenAlex × Crossref 双源 + ChinaXiv 兜底 + 摘要相关度（条件触发） |
| **v2（当前）** | **+ Step 0 Zotero 本地层（依赖用户全局 `mcp__zotero__*` MCP）+ 茉莉花富集元数据回读 + A/B/C 三态分流** |
| v3（候选） | + SinoMed / CNKI 网页解析（反爬可控时）+ 学科差异化策略 |

v2 改动只追加，不重写 v1：v1 网络层完整保留作 Zotero 未命中时的兜底层。
v2 新增文件：`REFERENCE_Zotero.md` / `RULES_zotero_layer.md` / `SETUP_Zotero.md` / `examples/case_zotero_complete.md` / `examples/case_zotero_stub.md`。

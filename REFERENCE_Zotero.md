# REFERENCE: Zotero 本地层（v2 核心信源）

> 本层依赖 `mcp__zotero__*` 工具族（由用户在 Claude Code 全局配置；本 skill 不自带 MCP 安装）。
> 自检与降级见 `SETUP_Zotero.md`。

## 工具族总览

| 用途 | 工具 | 关键参数 |
|---|---|---|
| 连通性自检 | `mcp__zotero__zotero_list_libraries` | 无参 |
| 关键词搜索（首选） | `mcp__zotero__zotero_search_items` | `query`（短）, `qmode=everything`, `limit=5` |
| 高级条件搜索（兜底） | `mcp__zotero__zotero_advanced_search` | `conditions=[{field,operation,value}]`, `join_mode=all` |
| 取完整元数据 | `mcp__zotero__zotero_get_item_metadata` | `item_key`, `format=json` 取原始字段 / `format=markdown` 看摘要 |
| 最近加入排查 | `mcp__zotero__zotero_get_recent` | `limit=20` 用于排查"刚加入未补全" |

## 查询策略（启发式）

1. **首选** `zotero_search_items`：
   - 有 DOI → `query=<DOI>`, `qmode=everything`（DOI 通常出现在 extra/字段中可被 substring 匹中）
   - 无 DOI → query 选标题里的显著名词/专有词 +（可选）第一作者姓，**总长 ≤ 8 字**
   - `qmode=everything` 默认，覆盖 title+abstract+creator+date
   - 重要：substring 匹配，多字会 narrow 结果，**不要塞整句标题**
2. **命中候选 ≥ 1 条** → `zotero_get_item_metadata(item_key=..., format="json")` 拿完整 data 对象
3. **未命中** → 退路一：`zotero_advanced_search` 用 `{field:"title", operation:"contains", value:"<某关键词>"}`；可与 `{field:"itemType", operation:"is", value:"journalArticle"}` AND
4. **仍未命中** → 状态 C·未命中，沉默通过本层

## 字段映射（Zotero `data.*` → 规范元数据）

| 规范字段 | Zotero `data.*` | 备注 |
|---|---|---|
| title | `title` | 直接取 |
| authors | `creators[].name` 或 `firstName + lastName` 拼接 | 中文条目常用单字段 `name` |
| journal | `publicationTitle` | |
| year | `date` 前 4 位 | 形如 `"2021-07-16"` / `"2021"` 都取 `2021` |
| volume | `volume` | |
| issue | `issue` | |
| pages | `pages` | |
| DOI | `DOI` | 注意：国内"DOI 风格内部编号"（如 `10.19891/...`）虽存于此字段也是合法记录，但**不在 Crossref**，要标注"国内编号" |
| abstract | `abstractNote` | |
| URL | `url` | |
| language | `language` | `zh` / `zh-CN` 是本 skill 适用条件 |
| 茉莉花溯源 | `extra` 行内的 `original-title` / `libraryCatalog` | 提示信源来自哪个中文库 |

## 「元数据完整」判据（决定 A 态 vs B 态）

**必要条件（全过即 A）：**
- `title` 非空
- `creators` 至少 1 项且至少有 `name` 或 `lastName`
- `date` 含 4 位年份
- `publicationTitle` 非空

**加分项（缺不降级）：** DOI / volume / issue / pages / abstractNote / url

**任一必要项缺失 → B 态（空壳·待补全）**

## 失败信号

| 现象 | 含义 | 动作 |
|---|---|---|
| `zotero_list_libraries` 报错 / 工具不存在 | MCP 不可用 | 整层跳过，报告顶部加配置提示 |
| `zotero_list_libraries` 通过但 item_count=0 | 不能据此判定库空（本地 SQLite 与 API 视图可能不同步） | 继续尝试 search |
| `zotero_search_items` 返回 0 条 | 关键词不匹配 | 走 advanced_search 二次尝试 |
| `advanced_search` 仍 0 条 | 库内无此条 | C·未命中，进网络层 |
| `zotero_get_item_metadata` 返 In Trash | 文献在回收站 | 视同未命中（不引用回收站条目） |

## 性能与轮次

- 单条文献预期：1 次 search + 0~1 次 advanced_search + 1 次 get_metadata = **2~3 次工具调用**
- 多条并行：单次 message 内并行 search，命中后再并行 get_metadata
- 不要对未命中的反复换关键词无限重试，最多 2 次 search + 1 次 advanced_search 即定 C 态

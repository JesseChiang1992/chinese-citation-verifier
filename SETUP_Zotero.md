# SETUP: Zotero MCP 配置说明（v2 依赖）

> 本 skill **不**自带 Zotero MCP 安装。Zotero MCP 是 Claude Code 的全局工具能力层，由用户单独配置；
> 本 skill 只负责消费 `mcp__zotero__*` 工具族。

## 自检（30 秒）

在任一 Claude Code 会话中调用：

- `mcp__zotero__zotero_list_libraries`

预期：
- 返回至少 1 个 user library（如 `My Library` / `我的文库`）
- item_count 字段可能为 0 或与客户端显示不同（本地 SQLite 视图与 API 视图不一定实时同步），**这正常**

调用报错或工具不存在 → MCP 未配置，跳过 Step 0。

## 配套：茉莉花插件（Zotero 客户端侧，**不是** MCP）

中文文献元数据富集依赖 Zotero 客户端内的**茉莉花**插件：
- 对接 CNKI / 万方 / 维普 / 中华医学杂志网 等中文信源
- 新条目自动补 DOI / 作者 / 期刊 / 卷期页 / 摘要
- 没装它，库里的中文条目会停留在「空壳」状态（仅 title），触发本 skill 的 B 态

茉莉花安装与配置不在本 skill 范围内，由用户在 Zotero 客户端处理。

## 没配置 / 临时失联时

| 场景 | skill 行为 |
|---|---|
| Zotero MCP 从未配置 | 跳过 Step 0；报告顶部加配置提示行；走 v1 网络层 |
| Zotero 客户端关闭中 | MCP 调用失败 → 同上降级 |
| 库内确实没存 | C 态，自然进网络层 |
| 存了但茉莉花未跑 | B 态，提示用户跑富集，**不自动**降级到 v1（防止低质数据污染） |

## 排错

- 调 `zotero_list_libraries` 通但 `zotero_search_items` 全报错 → 检查 Zotero 客户端是否处于"同步中 / 启动中"
- `zotero_get_item_metadata` 返 `Status: In Trash` → 文献在回收站，视同未命中
- search 命中但 item_count=0 且 list_libraries 也=0 → 多半是 local-mode 路径差异；search 命中本身就是连通证据，可继续走流程

# Case: Zotero 命中但元数据空壳（B 态）

> 以下为**示意**文献，演示 schema 与流程；实际查询时替换为真实输入。

## 输入

> 基于络病理论的慢性疼痛中医证治探讨

## Step 0 流程

1. `zotero_search_items(query="络病 慢性疼痛", qmode="everything", limit=5)`
   → 命中 `item_key=EFGH5678`（示意 key）
2. `zotero_get_item_metadata(item_key="EFGH5678", format="json")`
3. 字段判据：
   - title ✓ `基于络病理论的慢性疼痛中医证治探讨`
   - creators ✗ `[]`
   - date ✗ 空
   - publicationTitle ✗ 空
   - **B 态：空壳·待补全**

## 输出（暂停型）

~~~yaml
原始引用: 基于络病理论的慢性疼痛中医证治探讨
核验状态: 已存在·Zotero·待补全
信源命中: [Zotero]
zotero_item_key: EFGH5678   # 示意 key
缺失字段: [作者, 期刊, 年, 卷, 期, 页, DOI, 摘要]
建议:
  - 在 Zotero 客户端右键条目 EFGH5678 运行「茉莉花」插件富集元数据，富集完成后重试本核验（默认推荐）
  - 或：跳过 Zotero 层，让本 skill 走 v1 网络层（OpenAlex × Crossref；TCM 域命中率较低）
~~~

## 设计要点（铁律守护）

- title 已知 ≠ 可补字段。即便网络上能"找到"作者/期刊，**禁止**将查询结果填入规范元数据——这等同凭记忆补全（违反铁律三）
- **不**打 `未核验·信源失联`：文献确实在用户库里，只是茉莉花没跑
- **不**自动降级到 v1 网络层：避免国际索引在 TCM 域的低质命中污染元数据
- 默认走 (1) 路径：本条暂搁置，最终汇总单列"待茉莉花富集后重核"节
- 若用户在 Zotero 客户端中实际跑了茉莉花，本 skill 下次调用 `zotero_get_item_metadata` 会拿到富集后的字段，自动转 A 态

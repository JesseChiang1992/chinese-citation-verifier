# Case: Zotero 完整命中（A 态）

> 以下为**示意**文献，演示 schema 与流程；实际查询时替换为真实输入。

## 输入

> 针刺联合康复训练对脑卒中后偏瘫患者运动功能恢复的随机对照研究

## Step 0 流程

1. `zotero_search_items(query="针刺 脑卒中", qmode="everything", limit=5)`
   → 命中 `item_key=ABCD1234`（示意 key）
2. `zotero_get_item_metadata(item_key="ABCD1234", format="json")`
3. 字段判据：
   - title ✓ `针刺联合康复训练对脑卒中后偏瘫患者运动功能恢复的随机对照研究`
   - creators ✓ 多位（张某某, 李某某, 王某某 等）
   - date ✓ `2023-XX-XX`
   - publicationTitle ✓ `中华XX杂志`
   - **A 态：完整命中**，跳过网络层

## 输出

~~~yaml
原始引用: 针刺联合康复训练对脑卒中后偏瘫患者运动功能恢复的随机对照研究
核验状态: 已核验·Zotero·完整
信源命中: [Zotero]
zotero_item_key: ABCD1234   # 示意 key
规范元数据:
  标题: 针刺联合康复训练对脑卒中后偏瘫患者运动功能恢复的随机对照研究
  作者: [张某某, 李某某, 王某某, ...]
  期刊: 中华XX杂志
  年: 2023
  卷: XX
  期: X
  页: XXX-XXX
  DOI: 10.0000/example.cma.2023.001   # 示意 DOI
摘要: "目的 评价针刺联合康复训练对脑卒中后偏瘫患者运动功能恢复的影响。方法 ... 结果 ... 结论 ..."
摘要相关度: 未做此项
~~~

## 设计要点

- 网络层（OpenAlex / Crossref）整段跳过——茉莉花已经将 CNKI 一手元数据补齐，再跑国际索引仅为"凑双源"，本 skill 不为标签求源
- 标签固定 `已核验·Zotero·完整`，**不**升格为 `双源`（茉莉花 ≠ 第二独立信源；技术上仍是单源）
- 中华医学会系（前缀 `10.3760/cma....`）多为真实国际 DOI，Crossref 可查；用户若要求"再 Crossref 复核 DOI"可单独补跑，差异作为备注
- 摘要直接从 `abstractNote` 取，不重组、不补写

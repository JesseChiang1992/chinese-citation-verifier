# ChinaXiv — 兜底（预印本）

## 实情

ChinaXiv（中国科学院科技论文预发布平台）**没有官方公开 REST API**，但有可解析的网页接口。

## 启用条件

仅在 **OpenAlex 与 Crossref 都未命中** 且引用文本含下列线索时启用：
- "预印本"
- "preprint"
- "chinaxiv" / "ChinaXiv"
- URL 形如 `chinaxiv.org/abs/...`

否则**跳过 ChinaXiv 兜底**，直接走"未核验"报告。

## 搜索

```
GET https://chinaxiv.org/abs/search.htm?searchword={title}
```

返回 HTML 列表页，需提取：
- 详情链接：`<a href="/abs/{paperId}">...`
- 标题（链接文本）
- 作者、提交日期（列表行中的副文本）

## 详情

```
GET https://chinaxiv.org/abs/{paperId}
```

提取：
- 标题
- 作者列表
- 提交日期 / 年
- 摘要（详情页有摘要段）
- DOI（ChinaXiv 给每篇预印本独立 DOI）

## 字段映射（→ 规范元数据 schema）

| 规范字段 | ChinaXiv 来源 |
|---|---|
| 标题 | 详情页主标题 |
| DOI | 详情页 DOI 区块 |
| 作者列表 | 详情页作者段 |
| 期刊 | 固定标"ChinaXiv 预印本"（**不是**期刊文献） |
| 年 | 提交日期年 |
| 卷 / 期 / 页 | 空（预印本无） |
| 摘要 | 详情页摘要段 |

## 失效处理

HTML 结构可能随时变化。若关键选择器找不到匹配：
- 标 `ChinaXiv: parse_failed`
- **禁止**降级猜测或用记忆补全
- 直接进"未核验·信源失联"报告

## 不做的事

- 不解析 ChinaXiv 全文 PDF（v1 范围外）
- 不做 ChinaXiv 内部跨论文引用追踪
- 不与 OpenAlex / Crossref 做字段冲突仲裁（兜底层只单源）

## 调用示例

```
WebFetch(
  url="https://chinaxiv.org/abs/search.htm?searchword=<title>",
  prompt="List up to 5 preprint entries; for each, extract: detail URL (/abs/<id>), title, first author, submission date"
)

# 取到 detail URL 后
WebFetch(
  url="https://chinaxiv.org/abs/{paperId}",
  prompt="Extract title, all authors, submission date, DOI, abstract"
)
```

## 状态标签

| 结果 | 状态 |
|---|---|
| 单源命中 | `已核验·单源·ChinaXiv` |
| 解析失败 / 不可达 | 视为未命中，进未核验 |

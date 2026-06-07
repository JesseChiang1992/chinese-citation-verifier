# 案例 A：DOI 双源命中（最理想路径）

## 输入

```
张三, 李四. 某中文研究. 中华骨科杂志, 2023, 43(5): 300-308. DOI: 10.3760/cma.j.cn121113-20221001-00001
```

## 执行流程

### Step 1：归一化
- 提取候选字段：
  - 标题：某中文研究
  - 第一作者：张三
  - 期刊：中华骨科杂志
  - 年：2023
  - 卷：43；期：5；页：300-308
  - DOI：10.3760/cma.j.cn121113-20221001-00001

### Step 2：信源查询（并行）

并行调用两个 WebFetch：

```
WebFetch(
  url="https://api.openalex.org/works/doi:10.3760/cma.j.cn121113-20221001-00001?mailto=<YOUR_EMAIL>",
  prompt="Extract title, DOI, authorships, primary_location.source.display_name, publication_year, biblio, abstract_inverted_index"
)
WebFetch(
  url="https://api.crossref.org/works/10.3760/cma.j.cn121113-20221001-00001?mailto=<YOUR_EMAIL>",
  prompt="Extract message.title[0], message.author[], message.container-title[0], message.published.date-parts[0][0], message.volume, message.issue, message.page, message.DOI"
)
```

### Step 3：仲裁
- 标题：两源一致 → 一致
- 第一作者：OpenAlex "张三" vs Crossref `family=张, given=三` → 合成"张三" → 一致
- 期刊：两源均"中华骨科杂志" → 一致
- 年/卷/期/页/DOI：全部一致

→ 全字段一致 → **已核验·双源**

### Step 4：兜底
- 跳过（已双源命中）

### Step 5：摘要相关度
- 用户未给论点 X → **跳过**

### Step 6：出报告

## 输出

```yaml
原始引用: 张三, 李四. 某中文研究. 中华骨科杂志, 2023, 43(5): 300-308. DOI: 10.3760/cma.j.cn121113-20221001-00001
核验状态: 已核验·双源
信源命中: [OpenAlex, Crossref]
规范元数据:
  标题: 某中文研究
  作者: [张三, 李四]
  期刊: 中华骨科杂志
  年: 2023
  卷: "43"
  期: "5"
  页: "300-308"
  DOI: 10.3760/cma.j.cn121113-20221001-00001
摘要相关度: 未做此项
```

## 要点

- DOI 直查是首选路径，最快、最准。
- 两源都返回时，**字段一致性**才是"双源核验"的本质，不是"两边都查到了"。
- 用户未给论点 → 摘要相关度跳过，不强行启动（避免无用功）。

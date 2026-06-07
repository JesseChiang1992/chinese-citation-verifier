# OpenAlex API — 调用细则

## Base URL
`https://api.openalex.org`

## 礼貌池（polite pool）

所有请求加 `?mailto=<YOUR_EMAIL>`（OpenAlex 限速更宽：~100 req/sec vs 匿名 ~10 req/sec）。

## 主要端点

### 1) 按 DOI 直查（首选）
```
GET https://api.openalex.org/works/doi:{DOI}?mailto=<YOUR_EMAIL>
```
返回单条 Work 对象。

### 2) 标题 + 作者搜索（无 DOI 时）
```
GET https://api.openalex.org/works
    ?search={title}
    &filter=author.display_name.search:{firstAuthor}
    &per-page=5
    &mailto=<YOUR_EMAIL>
```
取 `results[]` 前 5 条，按 `display_name` 与查询标题相似度排序。

### 3) 中文文献定向过滤
追加 `filter=language:chi`（ISO 639-2 中文代码）：
```
&filter=language:chi
```
注意：OpenAlex 的语言标注未必精确，不宜作硬过滤，建议作"加权偏好"使用。

## 字段映射（→ 规范元数据 schema）

| 规范字段 | OpenAlex 路径 |
|---|---|
| 标题 | `title` |
| DOI | `doi`（带 `https://doi.org/` 前缀，需剥离） |
| 作者列表 | `authorships[].author.display_name` |
| 第一作者 | `authorships[0].author.display_name` |
| 期刊 | `primary_location.source.display_name`（新 schema）或 `host_venue.display_name`（兼容） |
| 年 | `publication_year` |
| 卷 | `biblio.volume` |
| 期 | `biblio.issue` |
| 起页 | `biblio.first_page` |
| 止页 | `biblio.last_page` |
| 摘要 | `abstract_inverted_index` —— **需重构**（见下） |

## 摘要重构（abstract_inverted_index）

OpenAlex 把摘要存成"词 → 位置数组"形式（规避版权）。重构步骤：

```
positions = []
for word, idx_list in abstract_inverted_index.items():
    for idx in idx_list:
        positions.append((idx, word))
positions.sort(key=lambda p: p[0])
abstract = " ".join(word for _, word in positions)
```

中文摘要重构可能因分词粒度不同而出现空格异常，**显示前去掉中文字符之间的空格**。

## 失效信号

| 现象 | 标记 | 后续动作 |
|---|---|---|
| HTTP 4xx | `OA: not_found` | 仅在两源都未命中时触发 Step 4 兜底 |
| HTTP 5xx / 超时 | `OA: error` | 若 Crossref 也 error → 触发"信源失联即停" |
| `meta.count == 0` | `OA: no_match` | 同 not_found |
| 网络不可达 | `OA: unreachable` | 同 error |

## 调用示例（伪 WebFetch）

```
WebFetch(
  url="https://api.openalex.org/works/doi:10.3760/cma.j.cn121113-20221001-00001?mailto=<YOUR_EMAIL>",
  prompt="Extract title, DOI, authorships[].author.display_name, primary_location.source.display_name, publication_year, biblio (volume/issue/first_page/last_page), and abstract_inverted_index"
)
```

## 注意事项

- DOI 大小写不敏感，但比对前统一转小写。
- `host_venue` 在新 schema 中被 `primary_location.source` 取代，但旧字段仍返回；取值时优先用 `primary_location.source`。
- 中文期刊 `display_name` 可能是中文也可能是英文官名（取决于注册），**不直接当字符串比对**，走名称归一化。

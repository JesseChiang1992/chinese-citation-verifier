# Crossref API — 调用细则

## Base URL
`https://api.crossref.org`

## 礼貌池

`?mailto=<YOUR_EMAIL>` 进入 polite pool（限速更宽）。User-Agent 亦推荐加 `mailto:` 标识。

## 主要端点

### 1) 按 DOI 直查（首选）
```
GET https://api.crossref.org/works/{DOI}?mailto=<YOUR_EMAIL>
```
注意：DOI 直接放路径，**不要 URL 编码**斜杠（Crossref 接受原始斜杠）。

### 2) 综合检索（bibliographic）
```
GET https://api.crossref.org/works
    ?query.bibliographic={titleword+author+year}
    &rows=5
    &mailto=<YOUR_EMAIL>
```
适用：有标题+作者+年份但无 DOI。

### 3) 字段化检索
```
GET https://api.crossref.org/works
    ?query.title={title}
    &query.author={firstAuthor}
    &rows=5
    &mailto=<YOUR_EMAIL>
```

### 4) 限定文献类型
```
&filter=type:journal-article
```
中文文献常见类型：`journal-article`、`book-chapter`、`proceedings-article`。

## 字段映射

| 规范字段 | Crossref 路径 |
|---|---|
| 标题 | `title[0]` |
| DOI | `DOI`（已是干净的 DOI 字符串） |
| 第一作者 | 见下"中文作者注意" |
| 作者列表 | `author[]`，每项 `{given, family, sequence}` |
| 期刊 | `container-title[0]` |
| 年 | `published.date-parts[0][0]`；若无则尝试 `published-print` / `published-online` |
| 卷 | `volume` |
| 期 | `issue` |
| 起止页 | `page`（格式如 `"123-130"`） |
| ISSN | `ISSN[]` |
| 摘要 | `abstract`（如有，常含 JATS XML 标签需清洗） |

## 中文作者注意

Crossref 中文作者注册形式不统一：
- 情况 A：`family=张, given=三` → 第一作者 = "张三"
- 情况 B：`family=张三, given=""` → 第一作者 = "张三"
- 情况 C：`family=Zhang, given=San` → 拼音；与汉字版本做"拼音 vs 汉字"弱一致比对
- 情况 D：英文出版商把中文姓名整段塞 `family` → 同 B

合成第一作者：`{family}{given}`（中文连写）或 `{given} {family}`（英文空格），按是否含 CJK 字符决定。

## 摘要清洗（JATS 标签）

Crossref 摘要可能含 `<jats:p>...</jats:p>` 等 XML 标签。清洗规则：
- 去除所有 `<jats:*>` / `</jats:*>` 标签
- HTML 实体解码（`&amp;` → `&`）
- 折叠多空格

## 失效信号

| 现象 | 标记 |
|---|---|
| HTTP 404 | `CR: not_found` |
| `message.items == []` | `CR: not_found` |
| HTTP 5xx / 超时 | `CR: error` |
| 网络不可达 | `CR: unreachable` |

## 中文期刊覆盖度提醒

Crossref 收录依赖出版商是否注册 DOI。中文医学顶刊（中华医学会系列）多已注册；社科类、地方期刊缺口大。
**未命中 Crossref 不等于文献不存在**，可能只是该期刊未注册 DOI。

## 调用示例

```
WebFetch(
  url="https://api.crossref.org/works/10.3760/cma.j.cn121113-20221001-00001?mailto=<YOUR_EMAIL>",
  prompt="Extract message.title[0], message.author[], message.container-title[0], message.published.date-parts[0][0], message.volume, message.issue, message.page, message.DOI, message.abstract"
)
```

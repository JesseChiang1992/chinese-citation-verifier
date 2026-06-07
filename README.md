# chinese-citation-verifier

> 一个 [Claude Code skill](https://docs.claude.com/en/docs/claude-code/skills) — 中文学术文献的元数据核验、引用补全、声明级核查；遵循"查不到不补、信源失联即停"的诚信原则。
>
> A Claude Code skill for verifying Chinese-language academic citations, with a strict no-fabrication integrity gate.

英文姊妹篇：[`academic-integrity-gate`](#) (PubMed/英文医学方向；本 skill 处理中文方向)

---

## 来龙去脉 (Background)

### 英文文献的核验：已经被解决

英文医学文献的核验路径在 AI 时代已经相对成熟：

**PubMed**（美国国家医学图书馆 NLM 出品）作为全球公认的金标准信源 → AI 工具通过 **PubMed MCP**（或类似 API 客户端）直接查询 → 可以核实文献是否存在、作者是否正确、甚至做"原文是否真的支持某声明"的声明级核查。Claude Code 用户配上 PubMed MCP + 一个诚信原则的 skill（比如 `academic-integrity-gate`），英文场景下"AI 编造引用"的问题就基本不存在了。

### 中文文献的核验：没有对等方案

中文学术界**没有**一个公开可访问的对等金标准信源。这是个结构性问题，不是 AI 工具的缺陷：

- **CNKI（中国知网）/ 万方 / 维普 / SinoMed** —— 覆盖最全，但**无公开 API**，反爬严格，AI 工具不能直接调用
- **OpenAlex / Crossref** —— 提供免费开放 API，但对中文文献覆盖**远不均匀**：

| 领域 | OpenAlex × Crossref 估计覆盖 |
|---|---|
| 西医中文期刊（中华××杂志 / Science China 系列等有国际 DOI） | 70–80% |
| STEM / 工科中文期刊 | 50–70% |
| **中医药 / 中医正骨 / TCM** | **< 10%** —— 大量 CNKI 独家、未注册国际 DOI |
| 社科 / 人文中文期刊 | < 20% |
| 学位论文 / 会议论文 / 灰文献 | < 5% |

更糟的是，国内一些期刊用"DOI 风格"内部编号（前缀如 `10.19891`、`10.13862` 等），**长得像 DOI 但并未在 Crossref 注册**。AI 调用时 HTTP 404，容易让 AI 或用户误判"文献不存在 = 假引用"。

### 折中方案：用 Zotero 做"私人金标准"

既然没有公开对等信源，唯一可靠的中文核验链路 = **用户自建的 Zotero 个人库** + AI 通过 MCP 读取。具体来说：

1. **用户**自己用 Zotero 策展中文文献库
2. **[茉莉花（Jasminum）](https://github.com/l0o0/jasminum)** 插件自动从 CNKI / 万方 / 维普 / 中华医学杂志网富集元数据（DOI、作者、卷期页、摘要 …）
3. **[Zotero MCP server](https://github.com/54yyyu/zotero-mcp)** 让 Claude Code 等 AI 工具按需查询本地 Zotero
4. **本 skill**（chinese-citation-verifier）= AI 侧的行为契约：优先查 Zotero，命中即用、空壳暂停、未命中再退到 OpenAlex × Crossref 兜底；全程不允许凭记忆补全

这是个"私人金标准"模式 —— 数据策展由用户负责，AI 只消费、不编造。中文文献核验的稳定性取决于你自己的 Zotero 库有多全。

---

## 必需前提 (Prerequisites)

本 skill **强依赖**于你自己的 Zotero 知识库链路。装本 skill 之前，请先确认下列环境就绪：

| 组件 | 是否必需 | 说明 |
|---|---|---|
| **Zotero 7+** 桌面客户端 | **必需** | 你的本地文献库。Windows / macOS / Linux 均可。**使用本 skill 时 Zotero 客户端必须保持运行** |
| **Zotero 本地 API**（已默认） | **必需** | Preferences → Advanced → General → 勾选 *"Allow other applications on this computer to communicate with Zotero"* |
| **Zotero MCP server** | **必需** | AI 通过它读你的 Zotero 库。推荐上游：[54yyyu/zotero-mcp](https://github.com/54yyyu/zotero-mcp) |
| **Jasminum（茉莉花）** Zotero 插件 | **强烈推荐** | 让中文条目自动从 CNKI / 万方 / 维普等补全元数据。**没装它，中文条目就停留在"仅有 title 的空壳"状态**，本 skill 会标 *已存在·Zotero·待补全* 并暂停，要求你先跑茉莉花。仓库：[l0o0/jasminum](https://github.com/l0o0/jasminum) |
| **Claude Code** | **必需** | 本 skill 运行的容器 |
| **Better BibTeX** Zotero 插件 | 推荐 | 提供稳定的 citation key 用于跨工具引用 |

如果你**完全没配** Zotero MCP，本 skill 会自动降级到 v1 网络层（OpenAlex × Crossref + ChinaXiv 兜底），但在 TCM / 国内队列 / 灰文献等领域命中率会很低（< 10%）。**强烈建议先把 Zotero 链路搭好再用本 skill**。

---

## 如何搭建 AI ↔ Zotero 的关系 (Setup)

从 0 到 1 的完整步骤。本 skill 自己只是其中一环（第 5 步）。

### Step 1 — 装 Zotero 7+

从 [zotero.org](https://www.zotero.org/) 下载安装。建议留意一下数据目录位置（默认 `~/Zotero/`，也可在 Preferences → Advanced → Files and Folders 中自定义到容量大的盘）。

### Step 2 — 装 Jasminum（茉莉花）

从 [l0o0/jasminum releases](https://github.com/l0o0/jasminum/releases) 下载最新 `.xpi`。在 Zotero 里：

`Tools → Plugins → 右上角 ⚙️ → Install Plugin From File...` → 选 `.xpi` → 重启 Zotero。

配置（可选）：`Edit → Preferences → Jasminum`，按需选 CNKI 翻译路径、显示语种、文件命名规则等。

> **为什么必需？** 中文文献从 PDF / 题录导入 Zotero 时，原生 Zotero 翻译器只能识别有国际 DOI 的少数中文期刊。茉莉花专门对接 CNKI / 万方 / 维普 / 中华医学杂志网，对中文条目（尤其是 TCM 域）补全率显著高于原生。**没有茉莉花，本 skill 的"完整命中"率会大幅下降。**

### Step 3 — 开启 Zotero 本地 API

`Edit → Preferences → Advanced → General` → 勾选 **"Allow other applications on this computer to communicate with Zotero"**。

Zotero 会在 `http://localhost:23119/` 开放本地 REST API。这是 Zotero MCP server 跟 Zotero 客户端通信的通道。

> Windows 用户注意：浏览器测 `localhost:23119` 时如果显示 "拒绝连接"，可能是浏览器把 localhost 解析成 IPv6 `::1` 而 Zotero 只绑了 IPv4 `127.0.0.1`。MCP server（Python）走 IPv4 不受影响，这个浏览器测试假阳性可以忽略。

### Step 4 — 装 Zotero MCP server

推荐上游：**[54yyyu/zotero-mcp](https://github.com/54yyyu/zotero-mcp)**。

参考其 README 安装。Python 用户可用 `uv` 或 `pipx`：

```bash
uv tool install zotero-mcp-server[scite]
# 或
pipx install zotero-mcp-server
```

然后把 MCP server **手工注册**到 Claude Code 的 `~/.claude.json`（`zotero-mcp setup` 命令当前只识别 Claude Desktop，不直接支持 Claude Code）：

```json
{
  "mcpServers": {
    "zotero": {
      "command": "/absolute/path/to/zotero-mcp",
      "args": ["serve"],
      "type": "stdio",
      "env": {
        "ZOTERO_LOCAL": "true"
      }
    }
  }
}
```

`command` 用绝对路径以避免 PATH 依赖问题。Windows 上通常在 `C:\Users\<user>\.local\bin\zotero-mcp.exe`。

重启 Claude Code（或开新会话），新会话调用 `mcp__zotero__zotero_list_libraries` 应能返库列表。

### Step 5 — 装本 skill

```bash
git clone https://github.com/JesseChiang1992/chinese-citation-verifier.git ~/.claude/skills/chinese-citation-verifier
```

（Windows PowerShell 用户：`~/.claude/skills/` 对应 `C:\Users\<你的用户名>\.claude\skills\`）

重启 Claude Code 或起新会话，`chinese-citation-verifier` 会出现在 available skills 列表里。

### Step 6（可选）— 配置 OpenAlex / Crossref polite-pool 邮箱

打开下列文件，把 `<YOUR_EMAIL>` 占位符替换成你自己的邮箱（仅作为 API URL 的 `?mailto=` 参数使用，进入 polite pool 享受更高速率限制，**不存储、不上传任何其他地方**）：

- `REFERENCE_OpenAlex.md`
- `REFERENCE_Crossref.md`
- `examples/case_doi_hit.md`

### 30 秒自检

新会话调用：

- `mcp__zotero__zotero_list_libraries` — 应返回 1+ 个 library（即使 item_count=0 也行，本地 SQLite 视图差异，搜索仍可能命中）
- `mcp__zotero__zotero_search_items(query="<你库里有的关键词>", qmode="everything")` — 应命中

两个都通 = 链路就绪。详见 [`SETUP_Zotero.md`](SETUP_Zotero.md)。

---

## 工作架构 (Architecture)

```
触发 (中英文短语 / 自动嗅探)
    ↓
Step 0:  Zotero 本地层 (if mcp__zotero__* 可用)
         ├── A · 完整命中  → 标 [已核验·Zotero·完整]，跳过网络层
         ├── B · 空壳命中  → 暂停，提示用户跑茉莉花；禁止凭 title 补字段
         └── C · 未命中    → 沉默通过，进入 Step 1-3 网络层
    ↓
Step 1:  输入归一化（繁→简 / 全角→半角 / 中英文标点）
Step 2:  网络查询（OpenAlex + Crossref 并行）
Step 3:  字段级仲裁（一致 / 弱一致 / 冲突）
Step 4:  ChinaXiv 兜底（仅当引用含"预印本 / preprint"字样）
Step 5:  摘要相关度检查（仅当用户给出论点 X）
Step 6:  按条出报告
```

---

## 五条铁律 (Five Core Rules)

1. **唯一真相源** — 文献是否存在、字段是否正确，只承认信源查证结果。模型记忆 / 训练知识**不构成证据**
2. **多源仲裁** — 网络层需要 OpenAlex + Crossref 一致才算「双源」；单源命中必须显式标源
3. **查不到不补** — 任何字段真相源未给，**绝不**凭记忆或推测补全。打标「未核验·信源失联」
4. **显性标注** — 每条引用必须出现状态标签：`已核验·Zotero·完整` / `已存在·Zotero·待补全` / `已核验·双源` / `已核验·单源·{OA|CR|ChinaXiv}` / `冲突·{字段}` / `未核验·信源失联`
5. **信源失联即停** — 任一次真相源调用失败 / 超时 / 全空响应，停下报告，不得用记忆顶替继续输出

---

## 触发方式 (Invoke)

显式触发短语（任一即可）：

- 中文：`核验中文文献` / `查证中文引用` / `中文文献核查` / `补全中文元数据` / `这些中文文献是真的吗`
- 英文：`verify Chinese citations` / `Chinese citation audit`

保守自动嗅探也会触发：用户粘贴 ≥ 3 条中文文献候选 + 问核查类问题；或 AI 即将输出中文参考文献段落。

---

## 输出 Schema（每条引用）

```yaml
原始引用: <用户原文>
核验状态: 已核验·Zotero·完整          # state A
        | 已存在·Zotero·待补全        # state B
        | 已核验·双源                # OpenAlex + Crossref 一致
        | 已核验·单源·{OA|CR|ChinaXiv}
        | 冲突·{字段}                # 多源不一致
        | 未核验·信源失联            # 所有信源失联
信源命中: [Zotero | OpenAlex | Crossref | ChinaXiv]
zotero_item_key: <Zotero 命中时填>
规范元数据:
  标题:   ...
  作者:   [..., ...]
  期刊:   ...
  年/卷/期/页/DOI: ...
冲突字段:                          # 仅 状态=冲突 时
  <字段>:
    来源A: ...
    来源B: ...
摘要:    <可得时填>
摘要相关度: 支持 | 部分支持 | 不支持 | 摘要不可得 | 未做此项
摘要锚点: "<原文 1-2 句>"
```

---

## 不适用 (Not for)

- **英文文献** → 用姊妹篇 `academic-integrity-gate` + PubMed MCP
- 非学术中文随笔 / 创意写作
- 一次性提及某文献（杀鸡用牛刀）

---

## 文件结构

```
chinese-citation-verifier/
├── SKILL.md                       # 行为契约、五条铁律、工作流
├── REFERENCE_OpenAlex.md          # OpenAlex API + 字段映射 + 摘要重构
├── REFERENCE_Crossref.md          # Crossref API + 中文作者处理 + JATS 清洗
├── REFERENCE_ChinaXiv.md          # ChinaXiv 预印本兜底
├── REFERENCE_Zotero.md            # Zotero MCP 工具族 + 查询策略 + 字段映射
├── RULES_cross_validation.md      # 双源仲裁矩阵
├── RULES_abstract_check.md        # 摘要相关度（条件触发）
├── RULES_zotero_layer.md          # Zotero A/B/C 三态分流
├── SETUP_Zotero.md                # Zotero MCP 自检 + 降级路径
└── examples/
    ├── case_doi_hit.md            # 双源 DOI 命中
    ├── case_conflict.md           # 字段冲突（禁擅断）
    ├── case_not_found.md          # 信源失联（诚信门退出）
    ├── case_zotero_complete.md    # Zotero A 态（完整命中）
    └── case_zotero_stub.md        # Zotero B 态（空壳·待茉莉花）
```

---

## License

MIT — see [LICENSE](LICENSE).

---

## 致谢 (Acknowledgments)

- [Zotero](https://www.zotero.org/) — 个人文献库的瑞士军刀
- [l0o0/jasminum（茉莉花）](https://github.com/l0o0/jasminum) — 中文文献元数据富集插件
- [54yyyu/zotero-mcp](https://github.com/54yyyu/zotero-mcp) — Zotero MCP server 上游实现
- [OpenAlex](https://openalex.org/) / [Crossref](https://www.crossref.org/) / [ChinaXiv](https://chinaxiv.org/) — 免费开放学术信源
- [Anthropic Claude Code](https://docs.claude.com/en/docs/claude-code/) — 本 skill 运行的容器

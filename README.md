# chinese-citation-verifier

A [Claude Code skill](https://docs.claude.com/en/docs/claude-code/skills) for verifying Chinese-language academic citations: dual-source cross-validation (OpenAlex × Crossref + ChinaXiv preprint fallback), optional local Zotero integration, and a strict "no fabrication" integrity gate.

Sister skill to `academic-integrity-gate` (English/PubMed); this skill handles the Chinese side.

## Why this exists

English biomedical literature has PubMed (and an excellent MCP). Chinese-language academia does not have a single equivalent gold-standard source. International indexes (OpenAlex, Crossref) have *uneven* coverage of Chinese journals:

| Domain | Estimated coverage |
|---|---|
| Western medicine in Chinese-language journals (中华XX杂志 / Science China series) | 70–80% |
| STEM / engineering | 50–70% |
| **Traditional Chinese Medicine (TCM / 中医药 / 中医正骨)** | **< 10%** — mostly CNKI-exclusive, no international DOI |
| Social sciences / humanities | < 20% |
| Theses, conference proceedings, grey literature | < 5% |

For low-coverage domains, this skill optionally queries your local **Zotero** library, assumed to be populated by the [Jasminum (茉莉花)](https://github.com/l0o0/jasminum) plugin, which auto-enriches Chinese entries from CNKI / Wanfang / VIP / Chinese Medical Journal Network.

Across every layer, the same rule applies: **if no source has the data, the skill does not invent it.** It labels the citation as `unverified·source-unreachable` instead.

## Architecture

```
Step 0:  Zotero local layer  (if mcp__zotero__* tools are available)
         ├── A: complete hit       → skip network, output verified·Zotero·complete
         ├── B: stub hit (title only) → pause, prompt user to run Jasminum
         └── C: miss               → silently fall through

Step 1:  Normalize input
Step 2:  Network query (OpenAlex + Crossref, in parallel)
Step 3:  Field-level arbitration  (consistent / weak / conflict)
Step 4:  ChinaXiv fallback        (only when preprint signals present)
Step 5:  Abstract relevance check (only when user supplies a claim X)
Step 6:  Per-citation report      (status label + canonical metadata)
```

## Install

```bash
git clone https://github.com/<your-username>/chinese-citation-verifier.git ~/.claude/skills/chinese-citation-verifier
```

Restart Claude Code (or open a new session); the skill will show up in the available-skills list.

## Configure

### 1. Polite-pool email (recommended)

OpenAlex and Crossref grant higher rate limits to identified requests via a `?mailto=` query parameter. Replace the `<YOUR_EMAIL>` placeholder in these files with your email:

- `REFERENCE_OpenAlex.md`
- `REFERENCE_Crossref.md`
- `examples/case_doi_hit.md`

The email is used **only** as an outbound URL query parameter to those two APIs.

### 2. Zotero local layer (optional, v2)

This skill does **not** bundle a Zotero MCP server. It expects you to configure one at the Claude Code level — that single MCP then serves all your Zotero needs, not just this skill.

Recommended upstream: [54yyyu/zotero-mcp](https://github.com/54yyyu/zotero-mcp). Follow its install instructions.

On the Zotero client side:

1. Install Zotero 7+.
2. Preferences → Advanced → General → enable **"Allow other applications on this computer to communicate with Zotero"**.
3. For Chinese metadata enrichment, install the [Jasminum (茉莉花)](https://github.com/l0o0/jasminum) plugin.

When all three are in place, this skill auto-detects `mcp__zotero__*` tools and routes citations through the Zotero layer first. See [SETUP_Zotero.md](SETUP_Zotero.md) for a 30-second self-check.

When the Zotero MCP is unavailable, the skill silently falls back to the network layer and prints a one-line hint at the top of the report.

## Invoke

Explicit triggers (any of):

- `核验中文文献` / `查证中文引用` / `中文文献核查` / `补全中文元数据` / `这些中文文献是真的吗`
- `verify Chinese citations` / `Chinese citation audit`

Conservative auto-sniffing also fires when the user pastes ≥ 3 Chinese citation candidates with a verification-style question, or when the assistant is about to emit a Chinese-language references section.

## Output schema (per citation)

```yaml
原始引用: <user's verbatim citation text>
核验状态: 已核验·Zotero·完整          # state A
        | 已存在·Zotero·待补全        # state B
        | 已核验·双源                # OpenAlex + Crossref consistent
        | 已核验·单源·{OA|CR|ChinaXiv}
        | 冲突·{field}              # multiple sources disagree
        | 未核验·信源失联            # all sources unreachable
信源命中: [Zotero | OpenAlex | Crossref | ChinaXiv]
zotero_item_key: <set only when Zotero hit>
规范元数据:
  标题:   ...
  作者:   [..., ...]
  期刊:   ...
  年:     ...
  卷/期/页/DOI: ...
冲突字段:  # only when state = conflict
  <field>:
    来源A: ...
    来源B: ...
摘要:      <when retrievable>
摘要相关度: 支持 | 部分支持 | 不支持 | 摘要不可得 | 未做此项
摘要锚点:   "<verbatim 1-2 sentence excerpt>"
```

## Not for

- **English literature** — use the sister skill `academic-integrity-gate` with the PubMed MCP.
- Non-academic Chinese prose / creative writing.
- One-off mentions of a single citation (overkill).

## File layout

```
chinese-citation-verifier/
├── SKILL.md                       # behavior contract, five rules, workflow, hard rules
├── REFERENCE_OpenAlex.md          # OpenAlex API endpoints + field map + abstract reconstruction
├── REFERENCE_Crossref.md          # Crossref API endpoints + Chinese-author handling + JATS cleanup
├── REFERENCE_ChinaXiv.md          # ChinaXiv preprint fallback
├── REFERENCE_Zotero.md            # Zotero MCP tool family + query strategy + field map
├── RULES_cross_validation.md      # Dual-source arbitration matrix
├── RULES_abstract_check.md        # Abstract-relevance check (conditional)
├── RULES_zotero_layer.md          # Zotero A/B/C state dispatch
├── SETUP_Zotero.md                # Zotero MCP self-check + fallback paths
└── examples/
    ├── case_doi_hit.md            # dual-source DOI hit
    ├── case_conflict.md           # field conflict (no auto-resolve)
    ├── case_not_found.md          # source unreachable (gate exits cleanly)
    ├── case_zotero_complete.md    # Zotero state A
    └── case_zotero_stub.md        # Zotero state B (pending Jasminum)
```

## License

MIT — see [LICENSE](LICENSE).

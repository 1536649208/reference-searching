# Reference Curator

发现 · 筛选 · 清理 — A [Claude Code](https://claude.ai/code) skill for academic reference curation. Three modes: (1) **Collection Searching** — mine references from Zotero collection papers; (2) **Web Searching** — search OpenAlex/Crossref directly; (3) **Literature Audit** — review existing papers to identify irrelevant ones. Features aggressive token optimization (~50-100% savings per phase), PDF keyword pre-screening, multi-source dedup, Zotero library dedup via set comparison, and claude-read tag tracking.

三模式学术文献策展工具：(1) **Collection Searching** — 从 Zotero collection 参考文献中挖掘新论文；(2) **Web Searching** — 从 OpenAlex/Crossref 直接搜索；(3) **Literature Audit** — 审查已有文献相关性，识别无关论文。内置激进 token 优化，输出去重纯 DOI 清单。

## How it Works

```
┌──────────────────────────────────────────────────────────────┐
│  /reference-curator                                          │
│                                                              │
│  1. Collection Searching  → Zotero → keywords → refs → DOI   │
│  2. Web Searching        → OpenAlex/Crossref → keywords → DOI│
│  3. Literature Audit     → Zotero/PDF → relevance → cleanup  │
└──────────────────────────────────────────────────────────────┘
```

Topic vs outline is **auto-detected** — drop a file path (PDF/DOCX/MD) for outline mode, or type a topic description for topic mode. Each mode asks exactly **3 questions**:

| Mode | Q1 | Q2 | Q3 |
|------|----|----|----|
| Collection | Topic or outline path | Output directory | Zotero collection name |
| Web | Topic or outline path | Output directory | Zotero collection (dedup) |
| Audit | Topic or outline path | Output directory | Collection or folder path |

### Pipelines

**Collection Searching**: source discovery → title pre-check → PDF keyword pre-screen → reference extraction → title+abstract filtering → global dedup → library dedup → `dois_new.txt` + `pending.txt` → tag source articles with `claude-read`

**Web Searching**: keyword derivation → 8 OpenAlex Boolean queries → inverted-index abstract reconstruction → small-keyword title verification → dedup → library DOI set comparison → `dois_new.txt`

**Literature Audit**: load papers (Zotero API or `pdftotext -l 2`) → title keyword check → abstract deep-read (title-miss only) → `irrelevant.txt` + `borderline.txt` + `audit_report.txt`

## Output Files

| Mode | File | Content |
|------|------|---------|
| Collection | `dois_new.txt` | New DOIs not in library |
| | `pending.txt` | Source papers without PDF |
| Web | `dois_new.txt` | New DOIs not in library |
| Audit | `irrelevant.txt` | DOIs suggested for removal |
| | `irrelevant_detail.txt` | Title + absent keywords |
| | `borderline.txt` | Unjudgeable papers (no abstract) |
| | `unreadable.txt` | Scanned PDFs (folder mode only) |
| | `audit_report.txt` | Statistics summary |

## Prerequisites

- **Claude Code** CLI or IDE extension
- **[Zotero MCP Server](https://github.com/anthropics/zotero-mcp)** (zoteus) — for Zotero API access
- A Zotero library (PDF attachments optional for collection mode)

## Installation

```bash
git clone https://github.com/1536649208/reference-searching-skill.git
cp reference-searching-skill/SKILL.md ~/.claude/skills/reference-curator/SKILL.md
```

Or via the Claude Code skill marketplace:

```
/install reference-curator
```

## Token Optimization

| Tactic | Saving |
|--------|--------|
| `concise` + `top:true` API format vs `detailed` | ~60% |
| Title keyword pre-check before PDF fetch | ~50% |
| Pre-screen 3000 chars → ref extraction 8000 chars | ~70% |
| Skip papers with zero keyword hits | ~100% |
| Abstract only for borderline papers (<20%, batch-fetched) | ~80%+ |
| DOI set comparison vs per-item Zotero search | ~90% |
| OpenAlex Boolean + structured fields vs Crossref | ~50% |
| Audit: zero external API calls | 100% |

## Read Tracking

The skill tags processed source articles with `claude-read` (cross-collection) to prevent duplicate work across sessions.

## License

MIT © 2026

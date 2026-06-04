# reference-searching

A [Claude Code](https://claude.ai/code) skill for automated citation chaining and reference mining from your Zotero library.

给定一个 Zotero collection 和一个研究主题，自动遍历 collection 中的源文章，从参考文献列表中挖掘与主题相关的新论文，并输出去重、库内查重后的 DOI 清单。

## How it Works

```
Zotero Collection → 源文章发现 → PDF全文关键词预筛 → 参考文献提取
    → 标题过滤 → 摘要深度判读（仅边缘文献）→ 去重 → 库内查重
    → dois_new.txt + dois_in_library.txt
```

The skill implements aggressive token optimization:
- **~60%** savings from concise API responses
- **~70%** savings from keyword pre-filtering PDFs before extracting references
- **~80%** savings from abstract-only-on-ambiguous-papers strategy
- **~90%** savings from zero-cost source-article dedup

## Prerequisites

- **Claude Code** CLI or IDE extension
- **[Zotero MCP Server](https://github.com/anthropics/zotero-mcp)** (zoteus) — provides the Zotero tools this skill uses
- A Zotero library with PDF attachments for the source papers

## Installation

```bash
# Clone the repo
git clone https://github.com/1536649208/reference-searching.git

# Copy the skill to your Claude Code skills directory
cp reference-searching/SKILL.md ~/.claude/skills/reference-searching/SKILL.md
```

Or install via the Claude Code skill marketplace:

```
/install reference-searching
```

## Usage

In Claude Code, invoke the skill:

```
/reference-searching
```

The skill will guide you through 4 required inputs:

1. **Zotero collection name** — which collection holds your source papers
2. **Topic / keywords** — what topic should the references match
3. **Output directory** — where to save `dois_new.txt`, `dois_in_library.txt`, and `pending.txt`
4. **Number of papers to process** — how many source papers to mine

### Output Files

| File | Content |
|------|---------|
| `dois_new.txt` | New references not found in your library |
| `dois_in_library.txt` | References already in your library (FYI) |
| `pending.txt` | Source papers that couldn't be processed (no PDF) |

### Read Tracking

The skill uses the Zotero tag `claude-read` to track which source papers have been processed, preventing duplicate work across sessions.

## Token Optimization

This skill was designed with token budget consciousness. Key strategies:

| Strategy | Savings |
|----------|---------|
| `concise` API format for Phase 1 item scanning | ~60% |
| Keyword pre-filtering before full reference extraction | ~70% |
| Skip papers with zero keyword hits in full text | ~100% of irrelevant papers |
| Abstract lookups only for ambiguous borderline papers | ~80% |
| Source-article DOI comparison over per-item API search | ~90% |

## License

MIT © 2026

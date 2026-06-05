---
name: reference-searching
description: "Search references of papers in a Zotero collection for papers related to a specific topic, or use a review outline to find relevant references either from Zotero reference chains or via direct web search. Use when the user asks to find papers related to topic X from the references of papers in a Zotero collection/library, wants to do citation chaining / reference mining from their Zotero library, or provides a review outline file to find relevant references. Triggers include: \"search references in my Zotero\", \"find papers about X from my library's references\", \"从我的Zotero库里找XX相关的参考文献\", \"在我的XX collection里搜参考文献\", \"根据综述大纲找参考文献\", \"帮我在写综述时找相关文献\"."
---

# Reference Searching Skill

Two search modes: Zotero reference chain mining, or direct academic database search. Output: DOI lists for papers matching user topic/outline.

## Mode Detection

Ask first, never guess:

> 1. 主题模式 — 给主题找文献  2. 综述大纲模式 — 给大纲找文献

---

## Phase 0a: Topic Mode

### Step 0a-1: Collect (single message)

1. Source: **A) Zotero chain** (mine refs from collection) / **B) Web search** (OpenAlex/Crossref)
2. Topic description (e.g. "铝在电池热管理方面的应用")
3. Output directory
4. Zotero collection name (A only)
5. Number of papers to process (A only)
6. Dedup collection key — optional. If given, use collection DOIs for O(1) library dedup in Phase 5/W5. (Recommended for all modes to avoid library duplicates in output.)

### Step 0a-2: Route

- **A → Phase 1**
- **B → Phase 0a-W** (derive keywords from topic, same logic as 0b-4, then → Phase W)

---

## Phase 0a-W: Topic → Structured Keywords → Web Search

Same keyword derivation logic as Phase 0b-4, but input is a natural-language topic description instead of an outline file. Derive grand keyword (hard constraint) + 6–8 small keyword groups (max 12) from domain knowledge. Format as Phase 0b-5. After user confirmation → Phase W1.

---

## Phase 0b: Outline Mode — Structured Keywords (Grand + Small)

### Step 0b-1: Collect (single message)

1. Source: **A) Zotero chain** / **B) Web search** (OpenAlex/Crossref)
2. Outline file path (PDF/DOCX/PPTX/XLSX/MD/txt/EPUB/HTML)
3. Output directory
4. Zotero collection name (A only)
5. Dedup collection key — optional. Same purpose as Step 0a-1 item 6.

### Step 0b-2: Convert

```bash
markitdown "<file>" -o /tmp/reference_searching_outline.md
```
Fallback: `pipx install 'markitdown[all]'` if missing; `strings`/`cat` if pipx unavailable.

### Step 0b-3: Parse

Extract heading hierarchy (`#`–`####`) and body paragraphs (material names, process params, methods).

### Step 0b-4: Derive Grand + Small Keywords (CRITICAL)

**Principle**: Use model DOMAIN KNOWLEDGE to expand, not mechanically extract from text.

**Step 1 — Grand keyword (hard constraint)**: Infer from review title + chapter headings. Every returned paper MUST relate to this. Example: "Aluminum Based Materials for BTMS" → `aluminum, aluminium, Al`.

**Step 2 — Small keyword groups (directional constraint)**: Derive 6–8 groups (up to 12 for large outlines with 3–4+ extra knowledge domains) by material/concept, NOT by chapter. Groups ordered by importance. Each group = search terms connected by OR.

Small keywords come from domain knowledge: "高导热增强体" → `SiC, diamond, graphene, AlN, B4C, carbon fiber`; "界面热阻" → `Kapitza resistance, thermal boundary conductance, acoustic mismatch model`.

**Step 3 — User tags**: `[关键词: xxx, yyy]` in outline → top priority, merge into matching group.

### Step 0b-5: Confirm (CRITICAL)

Show hierarchical structure. Format:

```
▌Grand (ALL papers must match): aluminum, aluminium, Al
▌Small (≥1 group per paper, max 12):
  [1] BTMS applications → battery thermal management, BTMS, EV battery cooling...
  [2] Al alloy thermal fundamentals → thermal conductivity, thermophysical, alloying...
  ...
Operations: add/del/modify any term; request more groups; change grand keyword.
```

After confirmation, route: **Zotero chain → Phase 1** | **Web search → Phase W**.

---

## Phase W: Web Search Pipeline

For topic mode B or outline mode B. Skip Zotero, search databases directly.

### Phase W1: Build Queries

Each small keyword group → 1 query: `grand_keyword AND (group terms OR-connected)`. N groups → N queries. Use OpenAlex Boolean syntax (uppercase AND/OR, `%22` for phrase quotes). **≤8 terms per group OR-clause** to avoid URL length overflow.

Example:
```text
aluminum AND ("battery thermal management" OR BTMS OR "EV battery cooling")
aluminum AND ("matrix composite" OR AMC) AND (SiC OR AlN OR diamond OR graphene) AND ("interface thermal resistance" OR Kapitza)
```

### Phase W2: Search

**Pre-check**: `openalex-paper-search` skill available (for abstract inverted-index reconstruction). If missing, degrade to Crossref for abstracts.

**Tier 1 — OpenAlex** (primary, Boolean search, 240M+ works):

```bash
curl -s "https://api.openalex.org/works?search=<encoded>&per_page=50&filter=has_abstract:true,type:article|review,language:en,is_retracted:false,publication_year:>2014&sort=relevance_score:desc&select=id,display_name,publication_year,cited_by_count,doi,authorships,abstract_inverted_index,type&mailto=agent@kortix.ai"
```

Key filters: `has_abstract:true`, `type:article|review`, `language:en`, `is_retracted:false`, `publication_year:>2014` (override on request). Use `mailto=` for 10 req/s. Reconstruct abstracts from inverted index via Python (see openalex-paper-search skill).

**Pagination**: `per_page=50` fetches top-50 by relevance. For queries returning >50 results, paginate with `&page=2` etc. (cap at 10 pages / 500 results per query to bound token cost). If a group needs deeper coverage, note in Phase W6 report.

**Tier 2 — Crossref** (supplement): Only if OpenAlex returns <10 results for a group. Note: Crossref lacks Boolean support; degrade query to keyword OR-connection. Stricter Phase W3 verification needed for Crossref results.
```bash
curl -s "https://api.crossref.org/works?query=<keywords>&rows=20"
```

**Tier 3 — Semantic Scholar** (last resort, likely blocked): `WebFetch` on API if above tiers fail.

### Phase W3: Verify Small Keyword Hits + Annotate

Grand keyword already enforced in query. Do NOT re-verify grand.

1. **Hit check**: Case-insensitive word-boundary match each paper title against small keyword terms. For terms ≤2 characters (e.g., "Al"), use \b word-boundary matching to avoid false positives ("Algorithm", "General"). Longer terms → substring match acceptable. ≥1 group matched → keep. Zero groups → discard. Multi-group matches → record all. (→ Pitfall 10)
2. **Coverage annotation**: Tag each kept paper with matching group number(s) for Phase W6 stats.
3. **Borderline papers**: If title unclear, batch-fetch abstracts. Collect all borderline DOIs → single bulk OpenAlex DOI lookup or batch Crossref calls. **Never fetch abstracts one-by-one.** Priority: OpenAlex inverted-index → Crossref API → keep.
4. **Review priority**: Borderline `type:review` papers → prefer keep (naturally high overlap with review outlines).
5. **Principle**: 宁可宽进不可窄出. User makes final call.

**Zero-result fallback**: If OpenAlex + Crossref return 0 for a group, flag in report with suggestions (expand terms, relax AND conditions, mark as under-researched area). Don't block other groups.

### Phase W4: Dedup

Same as Phase 4. Key priority: DOI (lowercased) → normalized title (lowercase, strip punctuation, first 80 chars) → author+year+title prefix.

Merge all query results, then global dedup.

### Phase W5: Library Dedup + Output

**If user provided dedup collection** (fastest, O(1) per paper, zero extra API calls):
1. `zotero_whoami` → confirm access
2. `zotero_search_items(collectionKey, limit:100, concise)` → get all DOIs
3. Python `set` comparison against candidate DOIs → discard matches
4. Paginate if collection >100 items

**If no dedup collection** (Web search mode):
⚠️ Without a dedup collection, library dedup is best-effort only. Output may include papers already in Zotero. Strongly recommend providing a dedup collection key in Step 0a-1/0b-1.
1. Title keyword search (only if <10 candidates): `zotero_search_items(q="<3-5 distinctive words>", qmode="titleCreatorYear")` per candidate → match → discard
2. Default: if >10 candidates, skip individual checking, keep all. Flag in Phase W6 report: "N candidates not checked against library — may contain duplicates."

**If no dedup collection** (Zotero chain mode):
1. Source article DOI comparison (free) — if candidate DOI matches any Phase 1 source DOI → discard
2. Title keyword search (only if <10 candidates remain): `zotero_search_items(q="<keywords>", qmode="titleCreatorYear")`
3. Default: if >10 candidates, skip individual checking, keep all

**Output** (`mkdir -p "<output_dir>"` first):

| File | Content |
|------|---------|
| `dois_new.txt` | One DOI per line, all NOT in library |

No `pending.txt` (web search has no source articles).

### Phase W6: Report

- Mode: topic→web / outline→web
- Outline path (outline mode only)
- Grand keyword; small keyword group count & terms per group
- Query count; data source stats (OpenAlex N / Crossref N)
- Small keyword hit verification: excluded N
- Abstract review: matched N
- After dedup: N kept (N duplicates removed)
- Library dedup: N discarded
- Small keyword coverage stats per group (compact inline format):
  ```
  [1] BTMS → 8  [2] Al fundamentals → 5  [3] Fabrication → 3  ...
  ```
  (Inline numbers only — no ASCII bar art needed.)
- Recommendations: flag sparse groups, suggest generating more or expanding terms.

---

## Phase 1: Discover Source Articles

1. `zotero_whoami` → confirm access
2. `zotero_list_collections` → target collection key
3. `zotero_search_items(collectionKey, limit:100, concise, top:true)` — concise saves ~60% tokens vs detailed; `top:true` excludes child notes/attachments at API level (→ Pitfall 1). Check `totalResults`: if >100, paginate with `start` offset to fetch all items.
4. Filter: only items where `collections` contains target key AND `itemType` ∈ {journalArticle, conferencePaper, book}. (Most child items already excluded by `top:true` above; → Pitfall 2)
5. **CRITICAL**: `zotero_search_items(tag:"claude-read")` (NO qmode) → exclude already-processed papers (→ Pitfall 3). If all candidates processed, report and ask.
6. Scope: topic mode → user-specified count; outline mode (Zotero chain) → all unprocessed
7. Sort: review → survey → rest. Topic keywords NOT used in sort.
8. Present list. Tagged papers shown as "SKIPPED (已处理)".

Note: Web search modes skip Phase 1 entirely — no source articles.

---

## Phase 2: Extract References Per Paper

Keyword-guided pre-screening → targeted extraction. Never pull entire reference list blindly.

**Keywords source**:
- Topic mode (Zotero chain): Derive pre-screen keywords from topic. Before Phase 2, show simple keyword list for user confirmation:
  ```
  Pre-screen: aluminum, Al → BTMS, thermal conductivity, PCM, foam, corrosion. Confirm?
  ```
- Outline mode (Zotero chain): Use Phase 0b-5 confirmed grand + small keywords.

**Step 0 — Title pre-check** (token optimization): Before any API call, scan each source paper's title against keywords. Title clearly irrelevant (zero keyword hits) → skip paper entirely. Title clearly relevant (≥2 keyword hits in title) → skip pre-screen, go direct to Step 3. Ambiguous → proceed to Step 1. This avoids unnecessary `zotero_get_fulltext` calls.

**Step 1 — PDF check**: `zotero_get_item(include_children:true)` → find `itemType:"attachment"` + `contentType:"application/pdf"`.

**Step 2 — Pre-screen** (core token optimization): `zotero_get_fulltext` with keyword list, `max_passages:3, max_chars:3000`. No hits → paper irrelevant, skip entirely.

**Step 2b — Pass-rate guard**: After processing all papers, compute pre-screen pass rate (papers with hits / total processed). If >80%, keywords are too broad (match nearly every paper's body text) → warn user and suggest refinement. Without refinement, token optimization is defeated. (→ Pitfall 11)

**Step 3 — Extract references** (if hits from Step 2, or direct from Step 0): `zotero_get_fulltext(query:"References"` OR `"Bibliography"` OR `"Works Cited"` OR `"参考文献", max_passages:4, max_chars:8000)`. Query the reference section header in the paper's language. Fallback: if none found, try `query:"[1]"` to locate reference list by citation bracket patterns. If truncated, use `page_range` to fill missing pages. Parse for entries containing pre-screen keywords → record title + DOI.

**Step 4 — Inline title filter**: Filter during extraction, don't accumulate. Keep only entries clearly relevant to topic. Discard rest immediately.

**No PDF**: Record to `pending.txt` as `Title — DOI`.

Never call `zotero_import`.

---

## Phase 3: Two-Pass Filtering

#### Pass 1 — Title Quick-exclude

Coarse filter: title clearly irrelevant → discard. (Phase 2 Step 4 does this inline already.)

#### Pass 2 — Abstract Deep-read (borderline only, <20% of candidates)

**Borderline definition** (HARD constraint): A paper is borderline IFF (a) title matched **<2 small keyword terms** AND (b) paper is **NOT** a review/survey type. Papers with ≥2 keyword hits in title OR type:review → PASS straight to keep, no abstract needed. (→ Pitfall 13)

**Threshold enforcement**: Count borderline papers before fetching abstracts. If borderline count >20% of total candidates → warn user: "N/Total (X%) papers flagged as borderline — abstract review will consume significant tokens. Continue? [y/n]". Default to skip abstract review and keep all if user doesn't respond.

Abstract retrieval priority (**batch only, never one-by-one**):
1. In Zotero library → `zotero_get_fulltext` (abstract section, `max_chars:2000`)
2. Has DOI → collect all borderline DOIs → single bulk OpenAlex lookup or batch Crossref calls. Reconstruct abstracts via Python.
3. No DOI, has title → OpenAlex search or Semantic Scholar fallback
4. Otherwise → keep (宁可保留不可误杀)

Judgment: does title/abstract directly involve user's topic? Is methodology/material closely related? Default: keep.

---

## Phase 4: Dedup (CRITICAL — Mandatory)

Multi-source papers may repeat across source articles. Dedup before writing.

Keys (priority): DOI (lowercased) → normalized title (lowercase, strip punctuation, first 80 chars) → author+year+title prefix.

Maintain `seen` set. Compute key → in set → skip | not in set → add, keep.

---

## Phase 5: Library Dedup + Output

Papers already in Zotero → silently discard.

**Strategy** (zero-API-call priority):
1. Source article DOI comparison (free): candidate DOI == any Phase 1 source article DOI → discard
2. Title keyword search (only if <10 candidates): `zotero_search_items(q="<3-5 distinctive words>", qmode:"titleCreatorYear")`
3. Default: >10 candidates → skip individual check, keep all

Output (`mkdir -p "<output_dir>"`):

| File | Content |
|------|---------|
| `dois_new.txt` | DOIs NOT in library, one per line |
| `pending.txt` | Source articles without PDF: `Title — DOI` |

No `dois_in_library.txt`. Library matches are silently dropped.

---

## Phase 6: Tag Source Articles (CRITICAL — Zotero Chain Only)

Web search modes skip this Phase (no source articles).

```text
zotero_manage_tags(action:"add", tags:["claude-read"], item_keys:[...])
```

- Tag SOURCE articles (not found references)
- `claude-read` is global, cross-collection
- Forgetting this = duplicated work next run (→ Pitfall 3)

**Verification** (mandatory): After tagging, run `zotero_manage_tags(action:"list", q:"claude")` and confirm all processed item_keys appear. If any missing → re-tag. Silent API failures here cause duplicated work next session.

---

## Phase 7: Report & Self-Improvement

- Mode: topic/outline × Zotero-chain/web-search (4 combos)
- Outline mode: outline path, grand keyword, small keyword groups + terms
- Web search: query count, OpenAlex N / Crossref N, coverage stats
- Zotero chain: source articles processed, pending.txt count
- Common: extracted N, title-excluded N, abstract-matched N, after-dedup N (duplicates removed), library-dedup N, dois_new.txt N, output path
- Coverage stats (web search only — Zotero chain uses free-form pre-screening, no structured groups)

---

## Token Optimization (CRITICAL — Every Run)

| Source | Before | After | Saving |
|--------|--------|-------|--------|
| Phase 1 scan | `detailed` format | `concise` + `top:true` format | ~60% |
| Phase 2 title pre-check | API call per paper | title keyword scan → skip irrelevant | ~50% |
| Phase 2 ref extraction | 20000 chars full refs | pre-screen→8000 chars targeted | ~70% |
| Phase 2 irrelevant papers | forced extraction | keyword miss→skip | ~100% |
| Phase 3 abstract review | every paper | borderline only (<20%, batch-fetched) | ~80% |
| Phase 3 batch abstracts | N individual API calls | 1 bulk lookup | ~90% |
| Phase 5 library dedup | per-paper Zotero search | source-DOI or dedup-collection set comparison | ~90% |
| Phase W2 data source | single Crossref | OpenAlex Boolean + select fields | ~50% |
| Phase W6 report | ASCII bar chart | inline numbers | ~30% |
| Phase 0b outline mode | mechanical term extraction | domain-knowledge grand+small structured queries | accuracy ↑↑↑ |

**Core rule**: Screen before fetch. Ask: "Is this API call needed? Can existing info decide?"

---

## Self-Improvement (CRITICAL)

After each run, review errors:
1. New error type → add Pitfall below (N+1)
2. Existing Pitfall retriggered → strengthen prevention wording
3. Format: `### Pitfall N: <short>`, `**Happened**:`, `**Prevention**:`

---

## Common Pitfalls

### Pitfall 1: items mixed with attachments/children
**Happened**: 96 results, 48 were child attachments.
**Prevention**: Filter by `collections` containing target key. Items with empty `collections` + `itemType:"attachment"` = children.

### Pitfall 2: Blind trust in OpenAlex/external metadata
**Happened**: Truncated titles, missing abstracts, incomplete authors → low-quality filtering.
**Prevention**: Don't discard metadata-incomplete borderline papers. Supplement via WebSearch.

### Pitfall 3: Missing claude-read check (start) and tag (end)
**Happened**: Processed papers without checking existing tags; forgot to tag after completion.
**Prevention**: Phase 1 Step 5 mandatory before selection. Phase 6 mandatory after completion. Both required.

### Pitfall 4: Duplicate DOIs in output
**Happened**: Same reference cited by multiple source articles, output without dedup.
**Prevention**: Phase 4 mandatory. Global `seen` set by DOI→title→author+year. Report dedup count.

### Pitfall 5: Already-in-library papers in output
**Happened**: Found papers user already has → redundant output.
**Prevention**: Phase 5 library dedup. Silently drop matches. User only sees new papers.

### Pitfall 6: Phase 2 pulling entire reference lists → token explosion
**Happened**: 43 papers × 20000 chars = massive token waste, mostly irrelevant refs.
**Prevention**: Keyword pre-screen mandatory (`max_passages:3, max_chars:3000`). No hits → skip paper. Hits → extract refs section with `max_chars:8000`.

### Pitfall 7: `tag` param written as `q` param
**Happened**: `q:"claude-read"` + `qmode:"everything"` returned 0 results; 54 tagged papers existed.
**Prevention**: Phase 1 Step 5 MUST use `tag:"claude-read"` (not `q`, no `qmode`). Verify with `zotero_manage_tags(action:"list", q:"claude")` if suspicious.

### Pitfall 8: Pre-screen hits body text, not references → low DOI yield
**Happened**: Keywords hit Introduction/Discussion (inline citations like "Wang et al. [24]"), not reference list.
**Prevention**: Pre-screen = relevance check only, NOT DOI extraction. After hits, MUST separately pull reference section via `query:"References"` + `page_range`.

### Pitfall 9: markitdown unavailable → outline mode fails
**Happened**: .docx outline but markitdown CLI missing or incompatible Python.
**Prevention**: `which markitdown` before conversion. Missing → `pipx install 'markitdown[all]'`. No pipx → fallback to `strings`/`cat` for text extraction.

### Pitfall 10: Short acronym false positives in keyword matching
**Happened**: Grand keyword "Al" substring-matched "Algorithm", "General", "signal" → irrelevant papers kept.
**Prevention**: Phase W3 hit check: terms ≤2 characters → use word-boundary matching (`\bAl\b`). Longer terms → substring match acceptable.

### Pitfall 11: Overly broad keywords → pre-screen always passes → token waste
**Happened**: Grand keyword matched every source paper's body text (paper is about that topic). Pre-screen pass rate 100% → every paper went to full reference extraction → ~70% token savings defeated.
**Prevention**: Phase 2 Step 2b pass-rate guard: if >80% papers pass pre-screen, warn user and suggest keyword refinement. Without refinement, token cost approaches pre-optimization levels.

### Pitfall 12: Web search mode without dedup collection → library duplicates in output
**Happened**: Web search found 50 papers; no dedup collection provided; >10 candidates → skip individual check → 30 papers already in library leaked to output.
**Prevention**: Phase W5: if no dedup collection in web search mode, warn user explicitly. Flag in Phase W6 report how many papers were NOT checked against library.

### Pitfall 13: Phase 3 borderline threshold not enforced → abstracts fetched for all papers
**Happened**: Model judged every paper as "borderline" → fetched abstracts for all 200 candidates → ~80% token savings from "borderline only" strategy completely lost.
**Prevention**: Phase 3 Pass 2 hard borderline definition: <2 keyword hits in title AND not a review. Count before fetching. If >20%, warn user and default to keep-all without abstract review.

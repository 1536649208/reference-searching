---
name: reference-curator
description: "Three-mode academic reference curator: (1) Collection Searching — mine references from Zotero collection papers by topic or review outline; (2) Web Searching — search OpenAlex/Crossref directly by topic or review outline; (3) Literature Audit — review a Zotero collection or local PDF folder against a topic/outline to identify irrelevant papers. Features: PDF full-text keyword pre-screening, two-pass title+abstract filtering, multi-source dedup, Zotero library dedup via set comparison, claude-read tag tracking, batch API calls, aggressive token optimization (~60-90% savings per phase). Output: pure DOI list (dois_new.txt) for search modes; irrelevant.txt for audit mode. Triggers: \"search references in my Zotero\", \"find papers about X from my library's references\", \"从我的Zotero库里找XX相关的参考文献\", \"在我的XX collection里搜参考文献\", \"根据综述大纲找参考文献\", \"帮我在写综述时找相关文献\", \"审查collection\", \"audit my collection\", \"找出无关文献\", \"整理文献\", \"清理collection\", \"文献策展\"."
---

# Reference Curator

发现 · 筛选 · 清理 — 学术文献策展工具

## Mode Detection

Ask first:

> 1. Collection Searching — 从 Zotero collection 参考文献挖掘新论文 → Phase 0-col
> 2. Web Searching — OpenAlex/Crossref 直接搜索 → Phase 0-web
> 3. Literature Audit — 审查已有文献相关性，识别无关论文 → Phase R

---

## Phase 0-col: Collection Searching

Mine refs from Zotero collection. Same collection = source + dedup. 3 questions:

1. Topic OR outline path — auto-detect: file extension → Outline; plain text → Topic
2. Output directory
3. Zotero collection name

Processes ALL unprocessed (non-claude-read) papers by default.

**Route**: Outline → Phase 0b-2 (→ Phase 1) | Topic → Phase 0-topic-keywords (→ Phase 1)

---

## Phase 0-topic-keywords: Topic → Keywords

Shared by Collection & Web when input is a topic. Derive grand keyword + 6–8 small groups (max 12) from domain knowledge. Format + confirm same as Phase 0b-5. Then → Phase 1 (Collection) or Phase W (Web).

---

## Phase 0-web: Web Searching

Search OpenAlex/Crossref directly. Zotero collection = dedup only. 3 questions:

1. Topic OR outline path — auto-detect same as Phase 0-col
2. Output directory
3. Zotero collection name (dedup only)

**Route**: Outline → Phase 0b-2 (→ Phase W) | Topic → Phase 0-topic-keywords (→ Phase W)

---

## Phase 0-keywords: Outline → Keywords

Shared by Collection & Web when input is an outline file.

### Step 0b-2: Convert
`markitdown "<file>" -o /tmp/reference_searching_outline.md`
Fallback: `pipx install 'markitdown[all]'`; `strings`/`cat` if unavailable.

### Step 0b-3: Parse
Extract heading hierarchy (`#`–`####`) and body paragraphs.

### Step 0b-4: Derive Grand + Small Keywords (CRITICAL)
Use DOMAIN KNOWLEDGE, not mechanical extraction.

- **Grand** (hard constraint, from title + headings): Every paper MUST relate. e.g. "Aluminum BTMS" → `aluminum, aluminium, Al`
- **Small** (6–8 groups, max 12, by material/concept NOT chapter): Domain-derived. "高导热增强体" → `SiC, diamond, graphene, AlN, B4C, carbon fiber`; "界面热阻" → `Kapitza resistance, thermal boundary conductance, acoustic mismatch model`
- **User tags**: `[关键词: xxx]` in outline → merge into matching group

### Step 0b-5: Confirm (CRITICAL)
```
▌Grand: aluminum, aluminium, Al
▌Small: [1] BTMS → battery thermal management, BTMS...  [2] Fundamentals → thermal conductivity...
Operations: add/del/modify terms; more groups; change grand.
```
→ Collection → Phase 1 | Web → Phase W

---

## Phase R: Literature Audit

Zotero/PDF → relevance check → irrelevant DOI list. No external search.

### Step R-A: Collect (3 questions)
1. Topic OR outline path — auto-detect same as Phase 0-col
2. Output directory
3. Zotero collection name OR local folder path — auto-detect: path-like (`/Users/...`, `/Volumes/...`) → Folder; plain name → Zotero collection

### Step R-B: Load Papers

| | Zotero | Folder |
|---|---|---|
| List | `search_items(collectionKey, limit:100, detailed, top:true)`, paginate | `find -name "*.pdf"` |
| Filter | `itemType` ∈ {journalArticle, conferencePaper, book, bookSection, thesis, preprint} | — |
| Extract | title, DOI, abstract, item_key | `pdftotext -l 2` each → regex DOI `\b10\.\d{4,}/[-.\w()/]+`, first-text-line title |
| Edge | — | Zero text → `unreadable.txt` |

### Step R-C: Keywords
Topic → grand + 4–6 broad groups (fewer than search modes — relevance bar, not search recall). Outline → full Phase 0b-4. Confirm before proceeding.

### Step R-D: Judgment
1. **Title** (zero API): ≥1 keyword hit → ✅ RELEVANT. 0 hits → step 2.
2. **Abstract** (title-miss only): Zotero: `get_item` (stored abstract) → if empty `get_fulltext(query:"abstract", max_chars:2000)`. Folder: Step R-B text. Hit → ✅; No hit → ❌ IRRELEVANT; No abstract → ⚠️ BORDERLINE (→ borderline.txt)
3. **Sanity**: >90% RELEVANT → warn (keywords too broad).

### Step R-E: Output
| File | Content |
|------|---------|
| `irrelevant.txt` | DOIs (or filenames) to remove |
| `irrelevant_detail.txt` | Title + absent keywords |
| `borderline.txt` | Unjudgeable (no abstract), keep by default |
| `unreadable.txt` | [B only] Scanned PDFs |
| `audit_report.txt` | N total → N relevant / N irrelevant / N borderline |

No auto-removal. User decides.

---

## Phase W: Web Search Pipeline

Direct OpenAlex/Crossref search. No Zotero reference mining.

### W1: Build Queries
Each small group → 1 query: `grand AND (group OR-terms)`. ≤8 terms per OR-clause. OpenAlex Boolean (uppercase AND/OR). Encode with `urllib.parse.quote`.

### W2: Search
**Tier 1 — OpenAlex**: `curl "https://api.openalex.org/works?search=<encoded>&per_page=50&filter=has_abstract:true,type:article|review,language:en,is_retracted:false,publication_year:>2014&sort=relevance_score:desc&select=id,display_name,publication_year,cited_by_count,doi,authorships,abstract_inverted_index,type&mailto=agent@kortix.ai"`. Paginate up to 10 pages/500 results per query. Reconstruct abstracts from inverted_index via Python.

**Tier 2 — Crossref**: Only if OpenAlex <10 results. `curl "https://api.crossref.org/works?query=<keywords>&rows=20"`. No Boolean support; stricter W3 verification.

**Tier 3 — Semantic Scholar**: WebFetch fallback.

### W3: Verify Small Keyword Hits
Grand already in query — don't re-verify.
1. Regex match titles against small terms (≤2 chars → `\b` word boundary). ≥1 group → keep. 0 → discard. (→ Pitfall 10)
2. Tag each paper with matched group numbers.
3. Borderline (title unclear): batch-fetch abstracts via OpenAlex bulk DOI lookup. Never one-by-one. Review type → prefer keep.
4. 宁可宽进不可窄出. 0 results for a group → flag, don't block others.

### W4: Dedup
Same as Phase 4: DOI ↓ → normalized title ↓ → author+year+title. Global `seen` set.

### W5: Library Dedup + Output
Collection from Step 0-web-1 = dedup library.
1. `zotero_whoami` → `list_collections` → resolve key
2. `search_items(collectionKey, limit:100, concise)` → all DOIs
3. Python `set`: candidates ∩ library → discard. Paginate if >100.

Output: `dois_new.txt` (DOIs NOT in library). No `pending.txt`.

### W6: Report
Mode, outline path, grand keyword, group count. Query count, OpenAlex N / Crossref N. Verification: excluded N, abstract-matched N. Dedup: N kept (N dupes). Library: N discarded. Group coverage: `[1] BTMS→8 [2] Al→5 [3] Fab→3...`. Flag sparse groups.

---

## Phase 1: Discover Source Articles (Collection only)

1. `zotero_whoami` → `list_collections` → key
2. `search_items(collectionKey, limit:100, concise, top:true)` — concise saves ~60%; `top:true` excludes children (→P1). Paginate if totalResults >100.
3. Filter: `collections` contains key AND `itemType` ∈ {journalArticle, conferencePaper, book} (→P2)
4. **CRITICAL**: `search_items(tag:"claude-read")` (NO qmode) → exclude processed (→P3). All done? Report.
5. Scope: all unprocessed. Limit if user asks.
6. Sort: review → survey → rest. Show: "SKIPPED (已处理)" for tagged.

---

## Phase 2: Extract References (Collection only)

Keyword-guided pre-screening. Never pull entire reference list.

**Keywords**: Topic → derive + confirm. Outline → Phase 0b-5 confirmed keywords.

- **Step 0 — Title pre-check**: Scan source title. 0 hits → skip paper. ≥2 hits → skip to Step 3. Else → Step 1.
- **Step 1 — PDF check**: `get_item(include_children:true)` → find `itemType:"attachment"` + `contentType:"application/pdf"`.
- **Step 2 — Pre-screen**: `get_fulltext(keywords, max_passages:3, max_chars:3000)`. 0 hits → skip. (→P11)
- **Step 2b — Guard**: >80% pass rate → warn (keywords too broad). (→P11)
- **Step 3 — Extract refs**: `get_fulltext(query:"References"|"Bibliography"|"Works Cited"|"参考文献", max_passages:4, max_chars:8000)`. Fallback: `query:"[1]"`. Use `page_range` if truncated. Parse entries matching keywords → record title+DOI.
- **Step 4 — Inline filter**: Discard irrelevant during extraction.
- **No PDF → `pending.txt`**: `Title — DOI`.

Never call `zotero_import`.

## Phase 3: Abstract Filtering (Collection only)

Borderline only (<20%): title matched **<2 keywords** AND **NOT review**. Count first — >20% → warn, default keep-all. (→P13)
Batch-fetch abstracts: Zotero `get_fulltext(abstract, max_chars:2000)` → bulk OpenAlex DOI lookup → Crossref. Default: keep.

## Phase 4: Global Dedup (CRITICAL)

Keys: DOI↓ → normalized title↓ (lower, strip punct, first 80 chars) → author+year+title. `seen` set.

## Phase 5: Library Dedup + Output (Collection only)

Source collection = dedup. Priority: 1) Source DOI compare (free) 2) Collection DOI set compare (free) 3) Title search (if <10 remain) 4) Skip check (>10).

Output: `dois_new.txt` (DOIs NOT in library) + `pending.txt` (no-PDF source articles). Library matches silently dropped.

## Phase 6: Tag Source Articles (CRITICAL — Collection only)

`zotero_manage_tags(action:"add", tags:["claude-read"], item_keys:[...])` — tag SOURCE articles only. Verify: `manage_tags(action:"list", q:"claude")` → confirm all keys present. (→P3)

---

## Phase 7: Report

Mode (Collection/Web/Audit × Topic/Outline), outline path, grand+small keywords. Search: queries, OpenAlex N / Crossref N, excluded N, abstract-matched N. Dedup: after-dedup N (dupes removed), library-dedup N. Output: dois_new.txt N, pending.txt N, output path. Audit: N→relevant/irrelevant/borderline. Coverage: web search groups; collection search free-form.

---

## Token Optimization

| Tactic | Saving |
|--------|--------|
| Phase 1: `concise`+`top:true` vs `detailed` | ~60% |
| Phase 2: title pre-check → skip irrelevant | ~50% |
| Phase 2: pre-screen→8000 chars vs 20000 chars | ~70% |
| Phase 2: keyword miss→skip paper entirely | ~100% |
| Phase 3: borderline only (<20%) vs all abstracts | ~80% |
| Phase 3: batch 1 lookup vs N individual calls | ~90% |
| Phase 5/W5: set comparison vs per-paper Zotero search | ~90% |
| Phase W2: OpenAlex + select fields vs Crossref | ~50% |
| Phase W6: inline numbers vs ASCII chart | ~30% |
| Phase 0b: domain-knowledge vs mechanical extraction | accuracy↑ |
| Phase R: title pre-check skips ~70% abstract reads | ~70% |
| Phase R: no external API calls | 100% |

**Core rule**: Screen before fetch.

---

## Self-Improvement

After each run: new error → add Pitfall (N+1); existing retriggered → strengthen wording. Format: `### Pitfall N: <short>`, `**Happened**:`, `**Prevention**:`.

---

## Common Pitfalls

### Pitfall 1: Children mixed with items
**H**: 96 results, 48 children. **P**: Filter by `collections` containing key; empty `collections`+attachment=child.

### Pitfall 2: Blind trust in external metadata
**H**: Truncated titles/abstracts → low-quality filtering. **P**: Don't discard metadata-incomplete borderline papers.

### Pitfall 3: Missing claude-read check/tag
**H**: Processed untagged papers; forgot to tag after. **P**: Phase 1 step 4 mandatory before; Phase 6 mandatory after.

### Pitfall 4: Duplicate DOIs in output
**H**: Same ref cited by multiple sources. **P**: Phase 4 mandatory. Global `seen` by DOI→title→author+year.

### Pitfall 5: Library duplicates in output
**H**: Web/Collection found papers already in Zotero → leaked to output. **P**: Collection mandatory; Phase W5/5 MUST DOI set-compare. Silent drop matches. No exceptions.

### Pitfall 6: Entire reference list pulled → token explosion
**H**: 43×20000 chars. **P**: Pre-screen `max_passages:3, max_chars:3000`. No hits→skip. Hits→refs `max_chars:8000`.

### Pitfall 7: `tag` param as `q`
**H**: `q:"claude-read"` + `qmode` → 0 results. **P**: MUST use `tag:"claude-read"` (NO qmode). Verify with `manage_tags(list, q:"claude")`.

### Pitfall 8: Pre-screen hits body, not references
**H**: Keywords matched inline citations, not ref list. **P**: Pre-screen=relevance only. MUST separately pull ref section: `query:"References"`+`page_range`.

### Pitfall 9: markitdown unavailable
**H**: .docx but no markitdown. **P**: `which markitdown` first. Missing→`pipx install 'markitdown[all]'`. Fallback: `strings`/`cat`.

### Pitfall 10: Short acronym false positives
**H**: "Al" matched "Algorithm". **P**: ≤2 chars → `\b` word boundary. Longer → substring ok.

### Pitfall 11: Overly broad keywords → pre-screen always passes
**H**: 100% pass rate → every paper full-extracted. **P**: Phase 2 Step 2b guard: >80% → warn.

### Pitfall 12: Borderline threshold ignored → all abstracts fetched
**H**: 200 papers all "borderline" → 200 abstract calls. **P**: Hard def: <2 keyword hits AND not review. >20%→warn, default keep-all.

### Pitfall 13: Audit per-paper abstract fetch
**H**: 50 title-miss→50 `get_fulltext` calls. **P**: Try `get_item` (stored abstract) first. Fallback to `get_fulltext`. Batch 5–10 with pauses.

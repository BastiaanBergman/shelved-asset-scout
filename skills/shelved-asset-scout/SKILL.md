---
name: shelved-asset-scout
description: |
  Surface de-prioritized ("shelved") pharmaceutical pipeline assets that may hold value
  for investors who specialize in acquiring, recycling, or licensing them.
  Cloud version — uses the PharmaceuticIndex MCP server at api.pharmaceuticindex.com.
  Requires a paid account. No local scripts or database needed.

  Use this skill when the user asks to:
  - Find de-prioritized, discontinued, shelved, or out-licensed pharma assets
  - Detect pipeline changes, communication cliffs, or strategic reprioritizations
  - Identify assets that have gone quiet in company press releases
  - Scout for pharma acquisition or licensing opportunities
  - Analyze pharmaceutical company press release patterns for investment signals
mcpServers:
  pharmaceuticindex:
    url: https://api.pharmaceuticindex.com/mcp
    authType: bearer
    authHint: "API key from pharmaceuticindex.com → Settings → API Keys"
---

# shelved-asset-scout

> Find big pharma's shelved assets before the market prices them in.

---

## Setup

This skill requires the **PharmaceuticIndex MCP server** connected to your AI client.

1. Create an account at **pharmaceuticindex.com** and subscribe to a paid plan
2. Generate an API key in your account dashboard under **Settings → API Keys**
3. Add the MCP server to your client configuration:
   - **Server URL:** `https://api.pharmaceuticindex.com/mcp`
   - **Auth:** Bearer token (your API key)
4. The server registers as `pharmaceuticindex` — all tools are prefixed `pharmaceuticindex__`

If a tool call returns an authentication error, ask the user to verify their API key and subscription status at pharmaceuticindex.com.

---

## Corpus Coverage

All features are available across the full corpus range **2020-01-01 → today**:

| Feature | Range |
|---|---|
| Press release text, title, summary | 2020-01-01 → today |
| Keyword search (`tsvector`) | 2020-01-01 → today |
| Therapeutic area tags (ltree) | 2020-01-01 → today |
| Modality tags (ltree) | 2020-01-01 → today |
| Semantic embeddings (Voyage) | 2020-01-01 → today |
| Company and date | 2020-01-01 → today |

Use semantic and tag search freely across any date range. Keyword search remains the preferred anchor for detecting multi-year silence arcs because it is exact-match and easy to reproduce; semantic search is best for confirmation and strategy-pivot detection.

---

## The Long-Arc Method (§7)

For detecting a company quietly dropping an asset over years:

**Step 1: Track the long arc**
- `pharmaceuticindex__get_activity_timeline` with `company` + `keyword` — tracks total company PR volume alongside keyword-matched mentions, continuous from 2020. A cliff in `filtered_count` while `total_count` stays healthy = selective silence = real signal.
- Use `pharmaceuticindex__get_corpus_volume_timeline` to rule out systemic gaps (joint dip in PR count + distinct_companies = ingestion artifact, not strategy).

**Step 2: Calibrate**
- `pharmaceuticindex__get_corpus_stats` with `company` — check cadence. A company with 3 PRs ever has no baseline; a gap is meaningless. Regular publishers (cadence_days < 60) have meaningful silences.

**Step 3: Confirm and zoom**
- `pharmaceuticindex__search_all_press_releases` with `mode: semantic` — high-resolution semantic search across the full corpus range.
- `pharmaceuticindex__list_therapeutic_areas` / `pharmaceuticindex__list_modalities` — see what TA/modality mix the company is publishing about; compare against keyword-detected activity to surface strategy pivots.

---

## Signal Taxonomy

Five strategies for flagging shelved assets. Each produces candidates with a **confidence rung** (low / moderate / high):

### 1. EOL / corporate-speak
Overt language of discontinuation or deprioritization:
- "strategic decision to focus resources on priority programs"
- "decision to discontinue development"
- "redirecting investment to core assets"

**Keyword search requires individual terms or short phrases — do not concatenate.** Run separate queries for each signal term; compound strings return zero hits. Validated high-signal terms (clean, low noise):

| Term | Signal quality | Notes |
|---|---|---|
| `discontinued` | High | Also matches "discontinuation" via stemming |
| `deprioritized` | High | Also matches "deprioritization" |
| `return of rights` | Medium | Also picks up financial accounting boilerplate; read snippets |
| `out-licensed` | High | Hyphenated form; catches explicit transfers |
| `restructuring` | Medium | Broad; filter by snippet for program-specific mentions |
| `focus resources` | Medium | Common in restructuring PRs; short phrase works |

**Avoid:** `"pipeline rationalization"` — tsvector stemming matches "rational pipeline" (false positives).

**Queries (run each separately):**
```
pharmaceuticindex__search_all_press_releases  query="discontinued"  mode=keyword  limit=50
pharmaceuticindex__search_all_press_releases  query="deprioritized"  mode=keyword  limit=50
pharmaceuticindex__search_all_press_releases  query="return of rights"  mode=keyword  limit=30
pharmaceuticindex__search_all_press_releases  query="out-licensed"  mode=keyword  limit=30
```

### 2. Communication cliff
An asset mentioned regularly that vanishes from the PR stream. Detected via `pharmaceuticindex__get_activity_timeline` with a keyword anchor on the asset/program name.

**Query:** `pharmaceuticindex__get_activity_timeline  company=X  keyword="asset-name"  period=month`

### 3. Asset outside corporate strategy
A TA/modality pivot that leaves an older program stranded. Company now publishing entirely about ADCs while older small-molecule oncology programs go silent. The semantic confirmation pass (Phase 2, Step 7) surfaces this automatically: if the semantic search for a shelved asset returns results about a *different* program or TA, that is the pivot signal.

### 4. Patent / LOE proxy
Lifecycle-management language as a proxy for approaching loss of exclusivity:
- "authorized generic"
- "label extension"
- "new formulation"
- "life cycle management"

**Note:** No hard patent dates in this corpus. This is a language-based proxy only.

**Queries (run separately):**
```
pharmaceuticindex__search_all_press_releases  query="authorized generic"  mode=keyword  limit=30
pharmaceuticindex__search_all_press_releases  query="life cycle management"  mode=keyword  limit=30
```

### 5. Out-licensing / return of rights
Explicit transfer or hand-back:
- "rights returned to"
- "out-license"
- "divest"
- "partner retains rights"

**Queries (run separately):**
```
pharmaceuticindex__search_all_press_releases  query="return of rights"  mode=keyword  limit=30
pharmaceuticindex__search_all_press_releases  query="out-licensed"  mode=keyword  limit=30
pharmaceuticindex__search_all_press_releases  query="divest"  mode=keyword  limit=30
```

---

## Gap vs. Silence — The Discriminator

Real deprioritization and scraping gaps both look like absence. Separate them by shape:

| Pattern | Interpretation | Confidence |
|---|---|---|
| Company still publishing (total activity normal), one asset vanishes | **Selective silence → real signal** | Moderate → High |
| Everything from a company disappears for a window, then resumes | **Total silence → likely scraping gap** | Low |
| Many unrelated companies show a gap in the same week/month | **Systemic ingestion artifact** | Low (rule out first) |

**The single most important check:** does `pharmaceuticindex__get_activity_timeline` show healthy `total_count` alongside a dropping `filtered_count`? If yes, the silence is asset-specific.

**Confidence ladder:**
- **Low**: total company silence → likely gap
- **Moderate**: selective silence + continued other activity
- **High**: selective silence + stale-but-current coverage + corroborating external signal (ClinicalTrials status change, IR page, web search)

**Confidence modifier — confirmed clinical failure:** If the asset failed a Phase 3 trial on its primary endpoint (not a safety halt, not a strategic pivot — a negative efficacy result), cap confidence at **moderate** regardless of silence quality. Strategic deprioritization ("resource allocation") and clinical failure ("the drug didn't work") carry categorically different acquisition risk. Annotate the reason in your candidate write-up.

---

## Tool Reference

All tools are provided by the `pharmaceuticindex` MCP server at `api.pharmaceuticindex.com`. Call them by their MCP tool names below.

---

### Navigation

#### `pharmaceuticindex__list_therapeutic_areas`
List TA ltree paths. Full range 2020-01-01 → today.

**Parameters:**
| Name | Type | Notes |
|---|---|---|
| `date_start` | string (YYYY-MM-DD) | optional |
| `date_end` | string (YYYY-MM-DD) | optional |
| `descendants_of` | string (ltree path) | optional — filter to subtree |
| `max_depth` | integer | optional |

**Output fields:** `paths` (list of `{path, pr_count, company_count}`), `total`, `coverage_window`

---

#### `pharmaceuticindex__list_modalities`
Same as `list_therapeutic_areas` but for `MOD.*` paths. Full range 2020-01-01 → today.

**Parameters:** same as `list_therapeutic_areas`

**Output fields:** `paths`, `total`, `coverage_window`

---

#### `pharmaceuticindex__list_companies`
List pharmaceutical companies. All filters operate across the full range 2020-01-01 → today.

**Parameters:**
| Name | Type | Notes |
|---|---|---|
| `ta` | string (ltree path) | optional |
| `modality` | string (ltree path) | optional |
| `keyword` | string | optional |
| `date_start` | string | optional |
| `date_end` | string | optional |

**Output fields:** `companies` (list of `{name, pcoid, pr_count}`), `total_companies`, `coverage_window?`

---

### Retrieval

#### `pharmaceuticindex__list_company_press_releases`
Browse press release titles and summaries for one company.

**Parameters:**
| Name | Type | Notes |
|---|---|---|
| `company` | string | **required** |
| `ta` | string (ltree path) | optional |
| `modality` | string (ltree path) | optional |
| `date_start` | string | optional |
| `date_end` | string | optional |

**Output fields:** `company`, `resolved_companies`, `records` (list of `{id, title, pdate, summary, company, tags}`), `total`, `coverage_window?`

Note: `tags` contains ltree paths for all records across the full date range.

---

#### `pharmaceuticindex__get_press_release`
Fetch full text for one or more press releases by ID.

**Parameters:**
| Name | Type | Notes |
|---|---|---|
| `ids` | list of integers | **required** |

**Output fields:** list of `{id, title, full_text, pdate, company, url, ta_paths, mod_paths}`

Note: `ta_paths` and `mod_paths` are populated for all records across the full date range.

---

### Search

#### `pharmaceuticindex__search_company_press_releases`
Search within a single company's press releases.

**Parameters:**
| Name | Type | Notes |
|---|---|---|
| `company` | string | **required** |
| `query` | string | **required** |
| `mode` | string | `keyword` / `semantic` / `tag` |
| `date_start` | string | optional |
| `date_end` | string | optional |
| `limit` | integer | optional, default 10 |

**Modes:**
- `keyword`: full-text search, full range 2020+
- `semantic`: Voyage embedding cosine similarity, full range 2020+
- `tag`: ltree path match (`query` is the ltree path), full range 2020+

**Output fields:** `company`, `mode`, `results` (list of `{id, title, pdate, snippet, score, chunk_index?}`), `total`, `coverage_window?`

---

#### `pharmaceuticindex__search_all_press_releases`
Corpus-wide search. Use when you don't yet know which company to target.

**Parameters:**
| Name | Type | Notes |
|---|---|---|
| `query` | string | **required** |
| `mode` | string | `keyword` / `semantic` / `tag` |
| `ta` | string (ltree path) | optional |
| `modality` | string (ltree path) | optional |
| `date_start` | string | optional |
| `date_end` | string | optional |
| `limit` | integer | optional, default 10 |

**Output fields:** `mode`, `results` (list of `{id, company, title, pdate, snippet, score, chunk_index?}`), `total`, `distinct_companies`, `coverage_window?`

---

### Temporal

#### `pharmaceuticindex__get_activity_timeline`
The absence/velocity engine. Shows total company activity alongside filtered series. **This is the primary tool for detecting communication cliffs.**

**Parameters:**
| Name | Type | Notes |
|---|---|---|
| `company` | string | **required** |
| `keyword` | string | mutually exclusive with `ta` |
| `ta` | string (ltree path) | mutually exclusive with `keyword`; full range 2020+ |
| `period` | string | `month` / `quarter` |
| `date_start` | string | optional |
| `date_end` | string | optional |

**Output fields:** `company`, `period`, `filter_mode`, `series` (list of `{period, total_count, filtered_count, past_cutoff}`), `coverage_window?`, `warning_note?`

**Note:** `keyword` mode is preferred for silence detection because it is exact-match and reproducible. `ta` filter mode is useful for TA-level pivots but produces broader matches; always corroborate with keyword before concluding absence.

---

#### `pharmaceuticindex__get_corpus_stats`
Calibration baseline. Always run this before interpreting silences.

**Parameters:**
| Name | Type | Notes |
|---|---|---|
| `company` | string | optional — omit for global stats |
| `date_start` | string | optional |
| `date_end` | string | optional |

**Output fields:** `keyword_coverage {start, end}`, `tag_coverage {start, end}`, `total_prs`, `distinct_companies`, `companies` (list of `{name, pr_count, first_date, last_date, cadence_days}`), `ta_counts`, `mod_counts`

`cadence_days` is `null` when a company has ≤2 PRs (no baseline).

---

#### `pharmaceuticindex__get_corpus_volume_timeline`
Systemic-gap detector. Run before concluding any batch of silences is real.

**Parameters:**
| Name | Type | Notes |
|---|---|---|
| `period` | string | `week` / `month` |
| `date_start` | string | optional |
| `date_end` | string | optional |

**Output fields:** `period`, `series` (list of `{period, pr_count, distinct_companies}`)

A dip in `pr_count` combined with a dip in `distinct_companies` = ingestion artifact. A dip in `pr_count` with stable `distinct_companies` = a real slow period.

---

### Presentation

#### `pharmaceuticindex__summarize_candidates`
Thin deterministic renderer for agent-assembled candidates. Does not query the database.

**Parameters:**
| Name | Type | Notes |
|---|---|---|
| `candidates` | list of candidate objects | **required** — see schema below |
| `no_matrix` | boolean | optional — omit strategy co-occurrence table |

**Candidate object fields:**

| Field | Type | Values |
|---|---|---|
| `company` | string | Company name |
| `asset_anchor` | string | Keyword/program name used to track this asset |
| `strategies` | list of strings | Any subset of the five strategy keys (see below) |
| `confidence` | string | `"low"` \| `"moderate"` \| `"high"` |
| `silence_class` | string | `"selective"` (asset gone, company still publishing) \| `"total"` (company gone too, likely gap) \| `"none"` (asset still visible — active out-licensing solicitation) |
| `cadence_class` | string | `"regular"` \| `"sparse"` \| `"no-baseline"` |
| `last_mention_date` | string or null | ISO date of last mention |
| `hit_window` | string | `"semantic"` (found via semantic/tag search) \| `"keyword"` (found via keyword timeline) \| `"both"` (confirmed by both methods) |

**Strategy keys:**
- `"eol_corporate_speak"`
- `"communication_cliff"`
- `"asset_outside_strategy"`
- `"patent_loe_proxy"`
- `"outlicensing_return_of_rights"`

**Output sections:** coverage banner, strategy scoreboard (with confidence breakdown + tier split), corroboration rollup, ranked candidate table.

---

## Standard Workflow

The workflow has two phases: **sweep** (find all candidates) then **confirm** (validate each one). Do not iterate per-company during the sweep — that wastes semantic quota on companies that never make the cut.

### Phase 1 — Sweep

**Step 1: Orient to the corpus.** Call `pharmaceuticindex__get_corpus_stats` (no arguments) to see date bounds, total PR count, and top TA/modality distribution. Note `cadence_days` for any companies that surface later.

**Step 2: Scan for direct signals.** Call each EOL keyword query separately — compound strings return zero hits:
```
pharmaceuticindex__search_all_press_releases  query="discontinued"  mode=keyword  limit=50
pharmaceuticindex__search_all_press_releases  query="deprioritized"  mode=keyword  limit=50
pharmaceuticindex__search_all_press_releases  query="return of rights"  mode=keyword  limit=30
pharmaceuticindex__search_all_press_releases  query="out-licensed"  mode=keyword  limit=30
```
Collect the (company, asset, date) tuples from snippets. A company appearing across multiple queries is a stronger candidate. These sweeps are exploratory — no need to preserve the raw results.

**Step 3: Check systemic gap risk.** Call `pharmaceuticindex__get_corpus_volume_timeline` with `period=month` and `date_start=2023-01-01`. Joint dips in `pr_count` + `distinct_companies` = ingestion artifact; those months cannot support absence claims.

**Step 4: For each candidate company, run an activity timeline.** Call `pharmaceuticindex__get_activity_timeline` with `company=X`, `keyword="asset-name"`, `period=quarter`. Is `total_count` healthy while `filtered_count` drops? → selective silence → proceed. Everything silent? → likely gap, skip or downgrade.

**Step 5: Check cadence.** Verify the company is a regular publisher (`cadence_days < 60` from `get_corpus_stats`). No baseline means no meaningful silence — skip or mark `cadence_class: "no-baseline"`.

**Step 6: Assemble draft candidates.** For each flagged asset, fill in all candidate object fields. Use `hit_window: "keyword"` if found via activity timeline / keyword sweep; upgrade to `"both"` after the semantic confirmation pass.

### Phase 2 — Confirm

**Step 7: Semantic confirmation pass (run all candidates in parallel).** For each candidate call:
```
pharmaceuticindex__search_company_press_releases  company=X  query="asset-name mechanism indication"  mode=semantic  limit=5
```
Two things to read from the results:
- **Is the asset absent?** No hits or very low scores → silence confirmed semantically → upgrade `hit_window` to `"both"`
- **What IS the company publishing about?** If the hits return a *different* program or TA, that is the `asset_outside_strategy` signal — add it to the candidate's strategies. This is how you detect strategy pivots without knowing the TA in advance.

**INN/renaming caution:** Programs often acquire INNs or rename between Phase 1 and Phase 2 (e.g. BT8009 → "pevedotin"; BT5528 → "nuzefatide pevedotin"). If semantic results return near-neighbor programs you don't recognise, check whether those are the same asset under a new name before concluding the shelved asset resurfaced.

**Step 8: Handle active out-licensing candidates separately.** If the semantic pass shows the asset is still appearing in company PRs (e.g. trial data being presented at a conference) but the company announced it is seeking partners, set `silence_class: "none"` and add `outlicensing_return_of_rights`. These candidates are often the **most immediately actionable** — the company is actively showcasing the asset to buyers — and should be highlighted as such in your write-up.

**Step 9: Apply confidence modifiers.** Review each candidate for the clinical failure modifier (see Gap vs. Silence section). Cap confirmed Phase 3 failures at moderate.

**Step 10: Render.** Call `pharmaceuticindex__summarize_candidates` with the assembled candidate list. Print the output to the conversation.

**Step 11: Write the narrative.** Present a **Signal Highlights** section (one paragraph per HIGH or MODERATE candidate explaining why it is interesting to an acquirer) and a **Caveats** section (confidence modifiers, ruled-out assets, any sweep-method limitations specific to this run).

---

## Worked Example

**Request:** "Find AstraZeneca assets that may have been quietly shelved since 2022."

```
# Step 1: Calibrate
pharmaceuticindex__get_corpus_stats  company="AstraZeneca"
# → cadence_days ~12 for AZ → regular publisher → silence is meaningful

# Step 2: Scan for direct signals (run separately)
pharmaceuticindex__search_company_press_releases  company="AstraZeneca"  query="discontinued"  mode=keyword  limit=20
pharmaceuticindex__search_company_press_releases  company="AstraZeneca"  query="deprioritized"  mode=keyword  limit=20
pharmaceuticindex__search_company_press_releases  company="AstraZeneca"  query="return of rights"  mode=keyword  limit=20

# Step 3: Check for systemic gaps
pharmaceuticindex__get_corpus_volume_timeline  period=month  date_start=2022-01-01
# → No joint dips → gaps are not artifacts

# Step 4: Track specific asset (e.g. "AZD-9291" from step 2)
pharmaceuticindex__get_activity_timeline  company="AstraZeneca"  keyword="AZD-9291"  period=quarter  date_start=2020-01-01
# → total_count 4-8/quarter through 2024, filtered_count drops to 0 from Q3 2024 → cliff

# Step 7: Semantic confirmation
pharmaceuticindex__search_company_press_releases  company="AstraZeneca"  query="AZD-9291 mechanism indication"  mode=semantic  limit=5
# → No results → asset absent from recent communications
# → Top hits are about different programs → asset_outside_strategy confirmed

# Step 10: Render
pharmaceuticindex__summarize_candidates  candidates=[{
  "company": "AstraZeneca",
  "asset_anchor": "AZD-9291",
  "strategies": ["communication_cliff", "eol_corporate_speak", "asset_outside_strategy"],
  "confidence": "high",
  "silence_class": "selective",
  "cadence_class": "regular",
  "last_mention_date": "2024-08-15",
  "hit_window": "both"
}]
```

---

## Constraints

**Do not:**
- Interpret `filtered_count = 0` on a TA-filtered timeline as proof of absence without corroborating with a keyword search — TA filters are broader and can miss assets
- Fabricate patent expiration dates — this corpus has no patent dates, only lifecycle-management language as a proxy
- Generate absence signals for companies with `cadence_days = null` or very high cadence — no baseline means no meaningful silence
- Conclude deprioritization from total silence without first checking `get_corpus_volume_timeline` for ingestion artifacts
- Silently pick one when a company name is ambiguous — the tools will error and list matches

**Always:**
- Check `total_count` alongside `filtered_count` in activity timelines
- State the confidence rung and tier-coverage caveat on every flag
- Call `pharmaceuticindex__summarize_candidates` at the end so the output is consistent and the coverage banner is visible

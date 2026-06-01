# PharmaceuticIndex MCP Server

> Pharmaceutical press release intelligence for AI agents — surface shelved assets, track pipeline silences, and detect strategic pivots across 2,000+ companies and 2020–present.

A hosted [Model Context Protocol](https://modelcontextprotocol.io) server at `https://api.pharmaceuticindex.com/mcp`. No local install required. Requires a paid account at [pharmaceuticindex.com](https://www.pharmaceuticindex.com).

---

## Installation

### Claude Desktop (one-click)

Click the button below to add the server to Claude Desktop automatically:

[![Install in Claude Desktop](https://img.shields.io/badge/Install_in-Claude_Desktop-blue?logo=anthropic)](claude://mcp/install?url=https%3A%2F%2Fapi.pharmaceuticindex.com%2Fmcp&name=pharmaceuticindex&authType=bearer)

Then open **Claude Desktop → Settings → MCP Servers**, find `pharmaceuticindex`, and paste your API key.

---

### Claude Code

Add to `~/.claude/settings.json`:

```json
{
  "mcpServers": {
    "pharmaceuticindex": {
      "url": "https://api.pharmaceuticindex.com/mcp",
      "authType": "bearer",
      "authToken": "YOUR_API_KEY"
    }
  }
}
```

Or use the CLI:

```bash
claude mcp add pharmaceuticindex \
  --url https://api.pharmaceuticindex.com/mcp \
  --auth-type bearer
```

You will be prompted for your API key.

---

### VS Code (Copilot / GitHub Copilot Chat)

Add to `.vscode/mcp.json` in your workspace, or to your user `settings.json` under `"mcp.servers"`:

```json
{
  "pharmaceuticindex": {
    "url": "https://api.pharmaceuticindex.com/mcp",
    "headers": {
      "Authorization": "Bearer YOUR_API_KEY"
    }
  }
}
```

---

### Other MCP-compatible clients

**Server URL:** `https://api.pharmaceuticindex.com/mcp`  
**Transport:** HTTP with SSE  
**Auth:** Bearer token (`Authorization: Bearer YOUR_API_KEY`)

---

## Getting an API key

1. Create an account at [pharmaceuticindex.com](https://www.pharmaceuticindex.com)
2. Subscribe to a paid plan
3. Go to **Settings → API Keys** and generate a key

---

## Skills

Skills are pre-packaged agent instructions that use this MCP server. Install a skill to get a purpose-built analyst, not just raw tool access.

### shelved-asset-scout (included in this repo)

Finds de-prioritized pharmaceutical pipeline assets — communication cliffs, discontinuation language, out-licensing signals — and scores them by confidence and signal type.

**Install:**
```bash
git clone https://github.com/bastiaanbergman/shelved-asset-scout.git \
  ~/.claude/skills/shelved-asset-scout
```

**Invoke:** `/shelved-asset-scout` in Claude Code

---

## Tools

All tools are prefixed `pharmaceuticindex__` when called via MCP.

| Tool | What it does |
|---|---|
| `search_all_press_releases` | Corpus-wide search — keyword, semantic, or ltree tag |
| `search_company_press_releases` | Same search scoped to one company |
| `list_company_press_releases` | Browse all PRs for a company with optional TA/modality filter |
| `get_press_release` | Fetch full text for one or more PRs by ID |
| `get_activity_timeline` | Company activity over time — total vs. filtered; the primary silence-detection tool |
| `get_corpus_stats` | Baseline calibration — PR counts, date bounds, cadence per company |
| `get_corpus_volume_timeline` | Systemic gap detector — weekly/monthly PR volume across all companies |
| `list_therapeutic_areas` | Browse TA ltree hierarchy (`TA.Oncology`, `TA.Cardiology`, …) |
| `list_modalities` | Browse modality ltree hierarchy (`MOD.Biologics.Antibody.Monoclonal`, …) |
| `list_companies` | List companies, filterable by TA, modality, or keyword |
| `summarize_candidates` | Render a ranked candidate table from agent-assembled results |

---

## Prompts

Copy any of these directly into your AI client with the MCP server connected.

---

**Find shelved assets at a specific company**
```
Using the pharmaceuticindex MCP, look for assets that [COMPANY] may have quietly 
de-prioritized or discontinued since [YEAR]. Check for communication cliffs, 
EOL language, and strategy pivots. Score each candidate by confidence.
```

---

**Scan a therapeutic area for out-licensing opportunities**
```
Using the pharmaceuticindex MCP, scan the [THERAPEUTIC AREA e.g. "TA.Oncology"] 
space for assets where companies have announced return of rights or out-licensing 
since 2022. Identify the most actionable candidates for a potential acquirer.
```

---

**Profile a company's pipeline communication patterns**
```
Using the pharmaceuticindex MCP, build a communication profile for [COMPANY]. 
Show me their total PR cadence, their main therapeutic areas and modalities, 
and flag any programs they were discussing regularly that have gone quiet 
in the last 18 months.
```

---

**Broad market sweep for deprioritization signals**
```
Using the pharmaceuticindex MCP, run a broad sweep across the corpus for 
discontinuation and deprioritization signals (search separately for "discontinued", 
"deprioritized", "return of rights", "out-licensed"). Identify the top 5 most 
interesting candidates for a specialty pharma acquirer and explain why each one 
is worth investigating.
```

---

**Verify a suspected silence**
```
Using the pharmaceuticindex MCP, I believe [COMPANY] has gone quiet on 
[ASSET/PROGRAM NAME] since around [DATE]. Confirm or rule out this silence: 
check their activity timeline, verify total company PR activity is healthy, 
look for any remaining mentions, and tell me whether this looks like selective 
silence or a data gap.
```

---

## Corpus coverage

| Feature | Date range |
|---|---|
| Press release text and title | 2020-01-01 → today |
| Keyword search (`tsvector`) | 2020-01-01 → today |
| Therapeutic area tags (ltree) | 2020-01-01 → today |
| Modality tags (ltree) | 2020-01-01 → today |
| Semantic embeddings (Voyage) | 2020-01-01 → today |

Companies covered: 2,000+ pharmaceutical, biotech, and CRO organizations.

---

## Building a new skill with this MCP

A skill is a Markdown file with YAML frontmatter that references this server. Minimum template:

```markdown
---
name: your-skill-name
description: |
  One-line description of what the skill does.
mcpServers:
  pharmaceuticindex:
    url: https://api.pharmaceuticindex.com/mcp
    authType: bearer
    authHint: "API key from pharmaceuticindex.com → Settings → API Keys"
---

# your-skill-name

Your skill instructions here. Reference tools by their MCP names:
pharmaceuticindex__search_all_press_releases, pharmaceuticindex__get_activity_timeline, etc.
```

Place the file at `.claude/skills/your-skill-name/SKILL.md`. See `shelved-asset-scout` 
in this repo for a full worked example including the two-phase workflow pattern, 
confidence scoring, and output rendering via `pharmaceuticindex__summarize_candidates`.

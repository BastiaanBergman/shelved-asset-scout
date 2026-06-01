# shelved-asset-scout

A Claude Code skill that surfaces de-prioritized pharmaceutical pipeline assets — communication cliffs, quiet discontinuations, and out-licensing signals — using the PharmaceuticIndex press release corpus.

## Requirements

- [Claude Code](https://claude.ai/code) installed
- A **premium account** at [pharmaceuticindex.com](https://pharmaceuticindex.com) with an API key

## Installation

1. Clone this repository into your Claude Code skills directory:

   ```bash
   git clone https://github.com/bastiaanbergman/shelved-asset-scout.git \
     ~/.claude/skills/shelved-asset-scout
   ```

2. Generate an API key at **pharmaceuticindex.com → Settings → API Keys**.

3. Add the PharmaceuticIndex MCP server to your Claude Code config (`~/.claude/settings.json`):

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

4. Restart Claude Code. The skill is available as `/shelved-asset-scout`.

## Usage

In any Claude Code session, type:

```
/shelved-asset-scout Find assets that Pfizer may have quietly shelved since 2023
```

The skill will run the full two-phase sweep-and-confirm workflow and present ranked candidates with confidence ratings.

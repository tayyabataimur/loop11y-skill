# loop11y-skill

[![skills.sh](https://skills.sh/b/tayyabataimur/loop11y-skill)](https://skills.sh/tayyabataimur/loop11y-skill)
[![npm](https://img.shields.io/npm/v/loop11y?label=loop11y%20mcp)](https://www.npmjs.com/package/loop11y)
[![License: MIT](https://img.shields.io/badge/License-MIT-purple.svg)](https://opensource.org/licenses/MIT)

The official Agent Skill for [Loop11y](https://github.com/tayyabataimur/loop11y) — accessibility evaluation, scoring, and remediation for any web product.

Teaches your coding agent (Claude Code, Cursor, Cline, Codex, OpenCode, Goose, GitHub Copilot, etc.) how to use the Loop11y MCP server correctly: when to call `evaluate`, when to escalate to `remediate`, how to handle auth, what to do when violations need manual fixes.

---

## Install

### 1. Install the skill

```sh
npx skills add tayyabataimur/loop11y-skill -g -a claude-code
```

Replace `claude-code` with your agent (`cursor`, `opencode`, `codex`, `cline`, etc.). See [supported agents](https://skills.sh/).

To install for all supported agents:
```sh
npx skills add tayyabataimur/loop11y-skill -g --all
```

### 2. Install the Loop11y MCP server

The skill orchestrates the [`loop11y`](https://www.npmjs.com/package/loop11y) MCP server. Add to your agent's MCP config:

```json
{
  "mcpServers": {
    "loop11y": {
      "command": "npx",
      "args": ["-y", "loop11y"]
    }
  }
}
```

Restart your agent.

---

## What it does

When the user mentions accessibility, a11y, WCAG, screen readers, ARIA, alt text, colour contrast, keyboard navigation, or shares a URL / component / repo for accessibility review, the agent:

1. Calls **`evaluate`** on the live URL → returns score (0–100), grade (A–F), WCAG level, ranked issues, AI summary
2. Offers **`remediate`** in `diff` mode if auto-fixable issues exist → previews patch
3. Applies fix on approval → writes safe changes to source
4. Re-runs `evaluate` to verify the score improved

For multi-page audits, the skill picks `crawl_site`. For codebase scans, `audit_repo`. For one-off rules, `fix_component`.

---

## Why a skill (and not just an MCP server)?

The MCP server gives the agent six tools. Without the skill, the agent often:
- jumps straight to `remediate` without auditing first
- forgets to ask for the source file path
- ignores severity filters and tries to fix everything
- skips the verify step after a patch
- tries to pass CLI auth flags through MCP (which doesn't support them)

The skill encodes the right workflow, default parameters, failure-mode handling, and tone guidance — so the agent gets it right every time.

---

## Files

```
skills/
└── loop11y/
    ├── SKILL.md           # workflow, tool selection, failure handling
    └── references/
        └── tools.md       # full input/output schemas
```

---

## Links

- [Loop11y MCP server + CLI on npm](https://www.npmjs.com/package/loop11y)
- [Loop11y main repo](https://github.com/tayyabataimur/loop11y)
- [Agent Skills spec](https://agentskills.io)
- [skills.sh directory](https://skills.sh)

## Licence

[MIT](./LICENSE)

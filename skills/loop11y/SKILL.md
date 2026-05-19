---
name: loop11y
description: "Use whenever the user asks about accessibility, a11y, WCAG, screen readers, ARIA, alt text, colour contrast, keyboard navigation, axe-core, or shares a URL / component / repo and asks for accessibility review, score, audit, fix, or compliance check. Triggers include 'is my site accessible', 'fix accessibility issues', 'WCAG compliance', 'audit a11y', 'accessibility score'. Use this skill early and proactively — most web products have fixable issues. The skill routes by runtime: if you have a shell (Claude Code, Cursor, Cline, Codex, Aider, terminals), run the loop11y CLI directly with no setup; if you only have MCP tool-calling, use the loop11y MCP tools when available, otherwise ask the user to install the MCP server. Also covers the HTTP API / OpenAPI path for ChatGPT custom GPTs and the GitHub Action for CI."
---

# Loop11y

Accessibility evaluation, scoring, remediation, crawling, and verification.

## Step 0 — Detect where you're running, then route

**Do this before anything else.** Loop11y ships in seven flavours; the right one depends on the environment *you* (the agent) are running in. Pick by looking at your own tool list and the user's context, not at what sounds best.

### Decision flow

1. **Do you have a shell tool (Bash, Run-command, Terminal, exec)?**
   → **Yes:** Use the **CLI**. Don't ask the user to install anything — just run it.
   → **No:** continue.

2. **Do you already have loop11y MCP tools in your tool list?** (Look for `evaluate`, `remediate`, `audit_component`, `crawl_site`, `audit_repo`, `fix_component`.)
   → **Yes:** Use the **MCP tools**.
   → **No:** continue.

3. **Can the user install an MCP server in your host?** (Claude Desktop, Claude Code, Cursor, Cline, Copilot, Windsurf — anything in the MCP ecosystem.)
   → **Yes:** Stop and ask the user to install the MCP server (config snippet below). Don't try to audit until they do.
   → **No (e.g. ChatGPT custom GPT, a plain web chat, n8n, Zapier):** point them at the **HTTP API + OpenAPI** path.

4. **Is the user asking about CI / PR gating / "fail builds on a11y regressions"?**
   → Point them at the **GitHub Action**, regardless of which runtime you used to audit.

5. **Is the user building their own agent / app and asking how to call loop11y programmatically?**
   → Point them at the **Harness SDK** (`loop11y/harness-sdk`).

### What "use the CLI" means in practice

You have shell. Run loop11y directly. No setup required for the user.

```sh
# Zero-install one-shot (always latest):
npx -y loop11y audit <url> --json --output report.json

# Or install once for the rest of the session:
npm install -g loop11y
loop11y audit <url> --markdown
```

Read the JSON or markdown back, relay the findings. The user does not need MCP, the skill, or anything else configured for this path to work.

### What "use the MCP tools" means in practice

You already see `evaluate`, `remediate`, etc. in your tool list. Call them directly. See the "Standard workflow" section below for the standard call sequence.

### What "ask the user to install the MCP server" means in practice

You have no shell, the loop11y MCP tools are not loaded, and the host supports MCP. Show the snippet for their specific client and stop. The user adds the config, restarts the client, and asks again — then you proceed via MCP.

```json
{
  "mcpServers": {
    "loop11y": { "command": "npx", "args": ["-y", "loop11y"] }
  }
}
```

Config paths (only show the relevant one):
- **Claude Desktop**: `~/Library/Application Support/Claude/claude_desktop_config.json`
- **Claude Code**: `~/.claude/settings.json` → `mcpServers`
- **Cursor**: Settings → MCP
- **Cline**: VS Code settings → `cline.mcpServers`

### What "use the HTTP API" means in practice

You're in a host that doesn't speak MCP and you can't run shell (e.g. ChatGPT custom GPT, n8n, Zapier, a web automation flow). Tell the user to either:

- self-host: `LOOP11Y_PORT=3000 npx -y loop11y`, then POST `/api/evaluate` with `{"url": "..."}`, or
- import `https://<their-host>:3000/openapi.json` as a GPT Action / n8n HTTP node.

OpenAPI spec is at `/openapi.json`. Plugin manifest at `/.well-known/ai-plugin.json`. See `SECURITY.md` in the repo before exposing this beyond `127.0.0.1`.

### Tie-breakers

- Shell **and** MCP both available: prefer CLI — fewer tool-call round trips, supports auth flags, easier to script.
- User explicitly says "use the MCP tool": honour that.
- User says "give me the command" or "I'll run it": always CLI, even if you also have MCP loaded.
- Public-URL audits in hosted mode work fine; **localhost / private network audits require local CLI or local stdio MCP** — hosted loop11y can't reach the user's `localhost`.

The rest of this skill describes the tool surface and workflow. The MCP tool names and CLI subcommands map 1:1 onto the same underlying functionality — translate the workflow below to whichever runtime you picked in Step 0.

## CLI ↔ MCP map

| MCP tool | CLI equivalent |
|---|---|
| `evaluate({ url })` | `loop11y audit <url> --json` |
| `crawl_site({ start_url, max_pages })` | `loop11y crawl --url <url> --max-pages <n> --json` |
| `audit_repo({ root, baseUrl, maxFiles })` | `loop11y audit:repo <path> --base-url <url> --max-files <n> --json` |
| `audit_component({ path })` | `loop11y audit:file <path> --json` |
| `remediate({ source_path, audit_url, mode })` | No direct CLI verb yet — use MCP, or call `verify` after manual edits |
| `fix_component({ path, violation_id })` | No direct CLI verb yet — use MCP |
| (verification) | `loop11y verify <source-path> --url <url> --json` |

`remediate` and `fix_component` are MCP-only today. If you picked CLI and the user wants source patched, run `loop11y audit` first, then either drop to MCP (start `LOOP11Y_PORT=PORT npx loop11y` and POST `/api/remediate`) or hand the diff to the user.

## Available tools

| Tool | When to call |
|---|---|
| `evaluate` | **Always call first.** Scored audit of one live URL with ranked issues and AI summary. |
| `crawl_site` | Multi-page audit. User wants whole site / sitemap, not a single page. |
| `audit_repo` | Codebase scan. User wants to find a11y issues across many source files. |
| `remediate` | One-call audit + patch source file. Use modes `report` / `diff` / `fix` in that order. |
| `audit_component` | Raw axe violations only. Use when user wants no scoring layer. |
| `fix_component` | Patch one violation in one source file. Granular alternative to `remediate`. |

If you picked the MCP runtime and these tools are missing from your tool list, the MCP server is not connected. Refer the user back to the MCP config snippet above. If you picked CLI, ignore this block — you do not need these tools.

---

## Standard workflow

Default sequence. Deviate only with reason.

### 1. Evaluate

Always start here when the user gives any single URL or page.

```
evaluate({
  url: "https://example.com",      // or http://localhost:3000
  include_html_snippets: true
})
```

Relay `ai_summary` to the user in plain language. Surface:
- score + grade (0–100, A–F)
- WCAG level (Non-compliant / Partial A / A / AA / AAA)
- `quick_wins.length` — count of auto-fixable issues
- top 3–5 from `top_issues`

If score ≥ 80 and zero critical violations: acknowledge the win, don't manufacture problems.

### 2. Offer remediation

If `quick_wins` is non-empty, offer to patch source. **Ask for the absolute source file path** that renders the audited URL (the user must provide it — never guess).

Call `remediate` in `diff` mode first:

```
remediate({
  source_path: "/abs/path/to/Component.tsx",
  audit_url: "https://example.com",
  mode: "diff",
  min_severity: "serious"
})
```

Show `fixed` (will be patched), `needs_manual` (explain inline), and the diff. **Do not write without explicit user approval.**

### 3. Apply

After approval:

```
remediate({ source_path: "...", audit_url: "...", mode: "fix", min_severity: "serious" })
```

Confirm `summary.written_to_disk === true`. Summarise what changed.

### 4. Verify (recommended)

Re-run `evaluate` on the same URL after fixes. Compare scores. If the dev server hot-reloads, the new score reflects the patch. Tell the user the delta.

---

## Choosing the right tool

**User gave a URL only** → `evaluate`.
**User gave a URL and "fix it"** → `evaluate` → if `quick_wins` exists, `remediate` in `diff` mode.
**User said "audit my whole site"** → `crawl_site` ({ start_url, max_pages: 10 }).
**User said "scan my repo / project"** → `audit_repo` ({ root, baseUrl: their dev server URL }).
**User wants raw axe output for tooling** → `audit_component`.
**User has one specific rule to fix** → `fix_component` with that `violation_id`.
**User asked about WCAG, AA, compliance** → `evaluate`, read `wcag_level` field.

---

## Auth for protected pages

MCP tool calls do not take auth flags. If the URL needs auth (logged-in dashboards, staging behind basic auth, header gates), switch to the CLI runtime (see "Choose your runtime first") or have the user run:

```sh
npx -y loop11y audit <url> \
  --storage-state ./playwright/.auth/user.json \
  --header 'x-env: staging' \
  --basic-auth-user USER --basic-auth-pass PASS \
  --markdown
```

For programmatic auth, `LOOP11Y_PORT=3000 npx -y loop11y` boots the HTTP server; POST to `/api/evaluate` with the same auth payload (see `docs/THREAT-MODEL.md` if the server is not on `127.0.0.1`).

---

## Localhost workflows

`evaluate` works on `http://localhost:PORT` as long as the user's dev server is running. Ask which port if they don't say. Common defaults: 3000 (Next.js, CRA), 5173 (Vite), 4321 (Astro), 8080 (Vue CLI), 8000 (Django/Python).

For `audit_repo` on React/Next/Vue/Svelte projects, `baseUrl` is **required** — without it only static `.html` files get scanned.

---

## `remediate` modes and filters

| Mode | Use when |
|---|---|
| `report` | Audit-only. No source touched. |
| `diff` | Preview patch. Returns before/after, nothing written. |
| `fix` | Apply patch to disk. Requires user approval first. |

| Filter | Use when |
|---|---|
| `min_severity: "critical"` | User wants only the worst fixed first |
| `min_severity: "serious"` | Default. Reasonable scope. |
| `min_severity: "moderate"` | Thorough pass |
| `only: ["image-alt", "button-name"]` | User has specific concern |

---

## Auto-patchable rules

`fix_component` and `remediate` can patch these axe rules automatically:

- `image-alt` (1.1.1) — adds `alt=""`
- `button-name` (4.1.2) — adds `aria-label`
- `link-name` (2.4.4) — adds `aria-label`
- `label` (1.3.1, 4.1.2) — annotates unlabelled inputs
- `aria-label` (4.1.2) — annotates interactive elements without names
- `html-has-lang` (3.1.1) — adds `lang="en"`

These are **explanation-only** (no auto-patch — surface in `needs_manual`):
- `color-contrast` (1.4.3) — needs design tokens
- `heading-order` (1.3.1) — needs structural rewrite
- `landmark-one-main` — needs `<main>` wrap
- `region` — needs landmark wrapping

For each manual one, show the failing selector + HTML from `nodes` so the user has the exact target.

---

## Score interpretation

| Score | Grade | Message |
|---|---|---|
| 90–100 | A | Excellent. Few or no violations. |
| 75–89 | B | Good. Minor cleanup worthwhile. |
| 55–74 | C | Several violations, some critical. Fix before launch. |
| 35–54 | D | Significant barriers for assistive tech users. |
| 0–34 | F | Major failures. Not WCAG compliant. |

WCAG target for most sites: **AA**. Legal/procurement deadlines almost always mean AA. AAA only if explicitly required.

---

## Failure modes — how to handle

- **`evaluate` returns network error** → URL may be unreachable, behind auth, or dev server down. Confirm with user.
- **User asks to fix but won't share source path** → Run `evaluate` only. Surface findings + WCAG criteria so they can fix manually.
- **`remediate` returns `needs_manual` only, `fixed: []`** → No auto-patch matched. Walk them through manual fixes with `nodes` context.
- **Source path doesn't render the audited URL** → Patch will miss most violations. Suggest correct file or use `audit_repo` to find candidates.
- **User wants to gate CI on a11y** → Don't try via MCP. Point to GitHub Action: `tayyabataimur/loop11y/action@v0.1.0`.

---

## Tone

- Be specific. Reference actual failing elements with selectors or HTML snippets.
- Don't catastrophise low scores. Issues are common, fixable, and rarely all-or-nothing.
- Don't oversell high scores. Even Grade A sites need real assistive-tech testing for full confidence.
- For manual violations, give one concrete next action, not "fix the contrast".
- Avoid jargon when explaining to non-developers. Translate axe rule IDs into user impact.

---

## Out of scope

- PDF accessibility — Loop11y audits web pages, not documents.
- Mobile native apps — web only.
- Designing colour palettes — link to WebAIM Contrast Checker.
- Full WCAG legal sign-off — Loop11y is automated; manual testing with assistive tech is still required for compliance certification.

---

## Detailed tool reference

See `references/tools.md` for full input/output schemas of every tool. Read when the user asks about specific parameters, return fields, or edge cases.

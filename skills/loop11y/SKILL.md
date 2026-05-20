---
name: loop11y
description: "Use whenever the user asks about accessibility, a11y, WCAG, screen readers, ARIA, alt text, colour contrast, keyboard navigation, axe-core, or shares a URL / component / repo and asks for accessibility review, score, audit, fix, or compliance check. Triggers include 'is my site accessible', 'fix accessibility issues', 'WCAG compliance', 'audit a11y', 'accessibility score'. Use this skill early and proactively — most web products have fixable issues. The skill routes by runtime: if you have a shell (Claude Code, Cursor, Cline, Codex, Aider, terminals), run the loop11y CLI directly with no setup; if you only have MCP tool-calling, use the loop11y MCP tools when available, otherwise ask the user to install the MCP server. Also covers the HTTP API / OpenAPI path for ChatGPT custom GPTs and the GitHub Action for CI."
---

# Loop11y

Accessibility, in a loop — **audit · share · fix · verify**. One engine, many surfaces: CLI, MCP server, GitHub Action, HTTP API, Harness SDK, web demo.

> Demo: https://loop11y.tayyaba.dev — paste URL, get audit. Useful when user wants to try before installing.

---

## The loop

```
audit  →  share findings  →  agent ships fix  →  re-audit (verify)
  ↑                                                      ↓
  └──────────────  score moves  ←───────────────────────┘
```

Always close the loop. After any `fix` / `remediate fix`, re-run `evaluate` (MCP) or `loop11y verify` (CLI) and report score delta. Don't claim success without verification.

---

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

OpenAPI spec at `/openapi.json`. Plugin manifest at `/.well-known/ai-plugin.json`. See `SECURITY.md` in the repo before exposing this beyond `127.0.0.1`.

### Tie-breakers

- Shell **and** MCP both available: prefer CLI — fewer tool-call round trips, supports auth flags, easier to script.
- User explicitly says "use the MCP tool": honour that.
- User says "give me the command" or "I'll run it": always CLI, even if you also have MCP loaded.
- Public-URL audits in hosted mode work fine; **localhost / private network audits require local CLI or local stdio MCP** — hosted loop11y can't reach the user's `localhost`.

The rest of this skill describes the tool surface and workflow. MCP tool names and CLI subcommands map 1:1 onto the same underlying functionality — translate the workflow below to whichever runtime you picked in Step 0.

---

## CLI ↔ MCP map

| MCP tool | CLI equivalent |
|---|---|
| `evaluate({ url })` | `loop11y audit <url> --json` |
| `crawl_site({ start_url, max_pages })` | `loop11y crawl --url <url> --max-pages <n> --json` |
| `crawl_site({ sitemap_url })` | `loop11y crawl --sitemap <url> --json` |
| `audit_repo({ root, baseUrl, maxFiles })` | `loop11y audit:repo <path> --base-url <url> --max-files <n> --json` |
| `audit_component({ path })` | `loop11y audit:file <path> --json` |
| `remediate({ source_path, audit_url, mode })` | No direct CLI verb — use MCP / HTTP `/api/remediate`, or call `verify` after manual edits |
| `fix_component({ path, violation_id })` | No direct CLI verb — use MCP / HTTP |
| (verification) | `loop11y verify <source-path> --url <url> --json` |
| (regression diff) | `loop11y diff <before.json> <after.json> --html --output diff.html` |

`remediate` and `fix_component` are MCP / HTTP only. If you picked CLI and the user wants source patched, run `loop11y audit` first, then either drop to MCP (`LOOP11Y_PORT=PORT npx -y loop11y` and POST `/api/remediate`) or hand the diff to the user.

---

## Available MCP tools

| Tool | When to call |
|---|---|
| `evaluate` | **Always call first.** Scored audit of one live URL with ranked issues and AI summary. |
| `crawl_site` | Multi-page audit. User wants whole site / sitemap, not a single page. |
| `audit_repo` | Codebase scan. User wants to find a11y issues across many source files. |
| `remediate` | One-call audit + patch source file. Use modes `report` / `diff` / `fix` in that order. |
| `audit_component` | Raw axe violations only. Use when user wants no scoring layer. |
| `fix_component` | Patch one violation in one source file. Granular alternative to `remediate`. |

Note: `verify` and `diff` exist in CLI / HTTP only — not MCP. To verify after MCP `remediate fix`, just call `evaluate` again on the same URL.

If you picked the MCP runtime and these tools are missing from your tool list, the MCP server is not connected. Refer the user back to the MCP config snippet above. If you picked CLI, ignore this block.

---

## CLI surface (full)

```bash
loop11y audit <url>               [--json|--markdown|--sarif|--html] [--output <file>]
loop11y audit:file <path>         [--json|--markdown|--sarif|--html] [--output <file>]
loop11y audit:repo <path>         [--max-files <n>] [--base-url <url>]
loop11y crawl --url <url>         [--max-pages <n>] [--include-pattern <re>] [--exclude-pattern <re>]
loop11y crawl --sitemap <url>     [--max-pages <n>]
loop11y crawl --routes <file>     [--max-pages <n>]
loop11y verify <source-path> --url <url>
loop11y diff <before.json> <after.json> --output report.html
```

Threshold gates (CI + scripting): `--fail-on critical|serious|moderate|minor`, `--max-violations <n>`, `--baseline <report.json>` for regression diffs.

Output formats:

| Flag | Use |
|---|---|
| `--json` (default) | machine-readable, feed back into `diff` / `--baseline` |
| `--markdown` | PR comment, Slack paste |
| `--html` | self-contained shareable visual report — best for non-devs |
| `--sarif` | GitHub Code Scanning, SonarQube |

Crawl auto-detects `/sitemap.xml`, `/sitemap_index.xml`, and `robots.txt` `Sitemap:` directives. Tracking params (`utm_*`, `gclid`, `fbclid`) stripped pre-dedupe. SPA route discovery is post-hydration via Playwright — Next.js / Remix routes are found.

---

## HTTP API (for ChatGPT GPT, n8n, Zapier, harness SDK)

Server: `LOOP11Y_PORT=3000 npx -y loop11y` (or Docker / Fly).

| Endpoint | Purpose |
|---|---|
| `POST /api/evaluate` | single URL audit |
| `POST /api/crawl` | site crawl |
| `POST /api/repo-audit` | scan a checked-out repo |
| `POST /api/remediate` | report / diff / fix source |
| `POST /api/verify` | re-audit after remediation |
| `POST /mcp` | streamable HTTP MCP transport |
| `GET /openapi.json` | OpenAPI 3 spec — wire into ChatGPT custom GPT Action |
| `GET /.well-known/ai-plugin.json` | plugin manifest |

Harness SDK:

```ts
import { Loop11yClient } from "loop11y/harness-sdk";
const client = new Loop11yClient({ baseUrl: "http://localhost:3000" });
await client.evaluate({ url: "https://example.com" });
```

---

## GitHub Action (CI gating)

User wants build to fail on regressions → never script this from MCP. Point them at:

```yaml
- uses: tayyabataimur/loop11y/action@v0.1.0
  with:
    url: https://staging.example.com
    fail-under: 90
```

Posts Markdown report as PR comment, sets check status. Score below `fail-under` fails the job.

---

## Standard workflow

Default sequence. Deviate only with reason.

### 1. Evaluate

Always start here when the user gives any single URL or page.

MCP:
```
evaluate({ url: "https://example.com", include_html_snippets: true })
```

CLI:
```sh
npx -y loop11y audit https://example.com --html --output report.html
```

Relay `ai_summary` in plain language. Surface:
- score + grade (0–100, A–F)
- WCAG level (Non-compliant / Partial A / A / AA / AAA)
- `quick_wins.length` — count of auto-fixable issues
- top 3–5 from `top_issues`

Score ≥ 80 with zero critical → acknowledge win, don't manufacture problems.

### 2. Offer remediation

If `quick_wins` non-empty, offer to patch source. **Ask for absolute source file path** that renders the audited URL (user must provide — never guess).

`remediate` in `diff` mode first:

```
remediate({
  source_path: "/abs/path/to/Component.tsx",
  audit_url: "https://example.com",
  mode: "diff",
  min_severity: "serious"
})
```

Show `fixed` (will be patched), `needs_manual` (explain inline), and the diff. **No writes without explicit user approval.**

### 3. Apply

After approval:

```
remediate({ source_path: "...", audit_url: "...", mode: "fix", min_severity: "serious" })
```

Confirm `summary.written_to_disk === true`. Summarise changes.

### 4. Verify (always)

Re-run `evaluate` on the same URL (MCP) or `loop11y verify <path> --url <url>` (CLI). Compare scores. If dev server hot-reloads, new score reflects the patch. Report the delta. **Loop closed.**

---

## Choosing the right tool

| User says | Action |
|---|---|
| URL only | `evaluate` / `loop11y audit <url>` |
| URL + "fix it" | `evaluate` → `remediate diff` → `remediate fix` → re-`evaluate` |
| "audit my whole site" | `crawl_site` / `loop11y crawl --url <url>` (or `--sitemap`) |
| "scan my repo" | `audit_repo` / `loop11y audit:repo <path>` (with `--base-url` for TSX/JSX/Vue/Svelte) |
| raw axe output | `audit_component` |
| one specific rule | `fix_component` with `violation_id` |
| "WCAG / AA compliance?" | `evaluate`, read `wcag_level` |
| "gate CI on a11y" | GitHub Action, not MCP |
| "share report with my manager" | CLI `--html --output report.html` |
| "compare against last week" | CLI `--baseline previous.json` or `loop11y diff before.json after.json --output report.html` |
| "send to Code Scanning" | CLI `--sarif --output a11y.sarif` |
| "try without installing" | https://loop11y.tayyaba.dev |

---

## Auth for protected pages

MCP tool calls do not take auth flags. If URL needs auth (logged-in dashboards, staging behind basic auth, header gates), switch to CLI or HTTP. CLI form:

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

`evaluate` / `audit` works on `http://localhost:PORT` as long as dev server is running. Ask the port if unsaid. Defaults: 3000 (Next.js, CRA), 5173 (Vite), 4321 (Astro), 8080 (Vue CLI), 8000 (Django/Python).

For `audit_repo` on React/Next/Vue/Svelte projects, `baseUrl` is **required** — otherwise only static `.html` files get scanned.

---

## `remediate` modes and filters

| Mode | Use when |
|---|---|
| `report` | Audit-only. No source touched. |
| `diff` | Preview patch. Returns before/after, nothing written. |
| `fix` | Apply patch to disk. Requires user approval first. |

| Filter | Use when |
|---|---|
| `min_severity: "critical"` | Worst-first |
| `min_severity: "serious"` | Default. Reasonable scope. |
| `min_severity: "moderate"` | Thorough pass |
| `only: ["image-alt", "button-name"]` | Specific concern |

---

## Auto-patchable rules

`fix_component` and `remediate` auto-patch these axe rules across `.tsx`, `.jsx`, `.html`, `.vue`, `.svelte`:

- `image-alt` (1.1.1) — adds `alt=""`
- `button-name` (4.1.2) — adds `aria-label`
- `link-name` (2.4.4) — adds `aria-label`
- `label` (1.3.1, 4.1.2) — annotates unlabelled inputs
- `aria-label` (4.1.2) — annotates interactive elements without names
- `html-has-lang` (3.1.1) — adds `lang="en"`

Explanation-only (surface in `needs_manual`, do not auto-patch):
- `color-contrast` (1.4.3) — needs design tokens
- `heading-order` (1.3.1) — needs structural rewrite
- `landmark-one-main` — needs `<main>` wrap
- `region` — needs landmark wrapping

For each manual one, show failing selector + HTML from `nodes` so the user has the exact target.

---

## Score interpretation

| Score | Grade | Message |
|---|---|---|
| 90–100 | A | Excellent. Few or no violations. |
| 75–89 | B | Good. Minor cleanup worthwhile. |
| 55–74 | C | Several violations, some critical. Fix before launch. |
| 35–54 | D | Significant barriers for assistive-tech users. |
| 0–34 | F | Major failures. Not WCAG compliant. |

WCAG target for most sites: **AA**. Legal / procurement deadlines almost always mean AA. AAA only if explicitly required.

---

## Determinism (when comparing runs)

Viewport pinned to 1280×800, `reducedMotion: reduce`, axe-core version emitted in every report. Same URL + same axe version = stable score. Use `--baseline` to assert no regressions.

---

## Failure modes — how to handle

- **`evaluate` returns network error** → URL may be unreachable, behind auth, or dev server down. Confirm with user.
- **User asks to fix but won't share source path** → Run `evaluate` only. Surface findings + WCAG criteria so they can fix manually.
- **`remediate` returns `needs_manual` only, `fixed: []`** → No auto-patch matched. Walk through manual fixes with `nodes` context.
- **Source path doesn't render the audited URL** → Patch misses violations. Suggest correct file or `audit_repo` to find candidates.
- **User wants CI gating** → Don't try via MCP. GitHub Action: `tayyabataimur/loop11y/action@v0.1.0`.
- **Vercel deploy of full loop11y** → won't work (Playwright + Chromium exceed serverless limits). Use Fly / Docker / local for engine; Vercel only for web frontend proxying to backend.

---

## Tone

- Be specific. Reference actual failing elements with selectors or HTML snippets.
- Don't catastrophise low scores. Issues are common, fixable, rarely all-or-nothing.
- Don't oversell high scores. Even Grade A sites need real assistive-tech testing for full confidence.
- For manual violations, give one concrete next action, not "fix the contrast".
- Avoid jargon when explaining to non-developers. Translate axe rule IDs into user impact.

---

## Out of scope

- PDF accessibility — Loop11y audits web pages, not documents.
- Mobile native apps — web only.
- Designing colour palettes — link to WebAIM Contrast Checker.
- Full WCAG legal sign-off — Loop11y is automated; manual assistive-tech testing still required for compliance certification.

---

## Detailed tool reference

See `references/tools.md` for full input/output schemas of every MCP tool, plus CLI command and HTTP endpoint reference. Read when user asks about specific parameters, return fields, or edge cases.

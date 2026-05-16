---
name: loop11y
description: "Use whenever the user asks about accessibility, a11y, WCAG, screen readers, ARIA, alt text, colour contrast, keyboard navigation, axe-core, or shares a URL / component / repo and asks for accessibility review, score, audit, fix, or compliance check. Triggers include 'is my site accessible', 'fix accessibility issues', 'WCAG compliance', 'audit a11y', 'accessibility score'. Use this skill early and proactively — most web products have fixable issues."
---

# Loop11y

Accessibility evaluation, scoring, remediation, crawling, and verification via the Loop11y MCP server.

## Available tools

| Tool | When to call |
|---|---|
| `evaluate` | **Always call first.** Scored audit of one live URL with ranked issues and AI summary. |
| `crawl_site` | Multi-page audit. User wants whole site / sitemap, not a single page. |
| `audit_repo` | Codebase scan. User wants to find a11y issues across many source files. |
| `remediate` | One-call audit + patch source file. Use modes `report` / `diff` / `fix` in that order. |
| `audit_component` | Raw axe violations only. Use when user wants no scoring layer. |
| `fix_component` | Patch one violation in one source file. Granular alternative to `remediate`. |

If these tools are missing from your tool list, the MCP server is not connected. Show:

```json
{
  "mcpServers": {
    "loop11y": { "command": "npx", "args": ["-y", "loop11y"] }
  }
}
```

Tell the user to add this to their MCP client config and restart.

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

If the URL needs auth (logged-in dashboards, staging behind basic auth, header gates), tell the user to run the CLI directly — MCP mode doesn't take auth flags. Give them:

```sh
npx loop11y audit <url> \
  --storage-state ./playwright/.auth/user.json \
  --header 'x-env: staging' \
  --basic-auth-user USER --basic-auth-pass PASS \
  --markdown
```

For programmatic auth they can also run `LOOP11Y_PORT=3000 npx loop11y` and POST to `/api/evaluate`.

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

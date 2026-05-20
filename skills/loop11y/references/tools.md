# Loop11y Tool Reference

Complete input/output documentation for all six MCP tools, plus the CLI and HTTP API surfaces.

> Tool naming: MCP exposes `evaluate`, `remediate`, `audit_component`, `fix_component`, `audit_repo`, `crawl_site`. CLI exposes `audit`, `audit:file`, `audit:repo`, `crawl`, `verify`, `diff`. HTTP exposes `/api/evaluate`, `/api/remediate`, `/api/repo-audit`, `/api/crawl`, `/api/verify`, `/mcp`. Schemas below describe MCP tool I/O; HTTP request/response bodies mirror them.

---

## `evaluate`

Audits a live URL and returns a scored accessibility report.

**Input:**
```ts
{
  url: string;                     // live URL or absolute file:// path
  include_html_snippets?: boolean; // include failing HTML from the page (default: true)
  include_passing?: boolean;       // include passing checks (default: false)
}
```

**Output:**
```ts
{
  url: string;
  timestamp: string;           // ISO 8601
  score: number;               // 0–100
  grade: "A" | "B" | "C" | "D" | "F";
  wcag_level: "AAA" | "AA" | "A" | "Partial A" | "Non-compliant";
  summary: {
    total_checks: number;
    passed: number;
    violations: number;
    critical: number;
    serious: number;
    moderate: number;
    minor: number;
    auto_fixable_count: number;
    incomplete: number;
  };
  top_issues: IssueSuggestion[];  // ranked by impact × node count
  quick_wins: IssueSuggestion[];  // auto-fixable violations only
  passing_checks?: string[];      // if include_passing: true
  ai_summary: string;             // plain-text narrative, relay this to the user
}
```

**IssueSuggestion:**
```ts
{
  rank: number;
  violation_id: string;          // axe rule ID (e.g. "image-alt")
  impact: "critical" | "serious" | "moderate" | "minor";
  wcag_criterion: string;        // e.g. "WCAG 1.1.1"
  affected_elements: number;     // number of failing nodes
  headline: string;              // e.g. "6 images are missing alternative text"
  user_impact: string;           // what real users experience
  suggestion: string;            // specific fix guidance
  example_before: string;        // failing HTML (from live page if snippets enabled)
  example_after: string;         // corrected HTML
  auto_fixable: boolean;         // can remediate patch this?
  learn_more: string;            // Deque University URL for the rule
}
```

---

## `remediate`

Audits a URL and patches the corresponding source file. All violations applied in a single chained pass.

**Input:**
```ts
{
  source_path: string;   // absolute path to .tsx / .jsx / .html to patch
  audit_url: string;     // URL to audit (rendered page)
  mode: "report" | "diff" | "fix";
  min_severity?: "minor" | "moderate" | "serious" | "critical"; // default: "serious"
  only?: string[];       // allowlist of violation IDs (optional)
}
```

**Output:**
```ts
{
  mode: string;
  audit_url: string;
  source_path: string;
  min_severity: string;
  summary: {
    total_violations: number;
    auto_fixed: number;
    needs_manual: number;
    skipped: number;
    written_to_disk: boolean;
  };
  fixed: Array<{
    violation_id: string;
    wcag: string;
    explanation: string;
    nodes_affected: number;
  }>;
  needs_manual: Array<{
    violation_id: string;
    impact: string;
    description: string;
    wcag: string[];
    reason: string;
    nodes: Array<{ selector: string; html: string; failureSummary: string }>;
  }>;
  skipped: Array<{
    violation_id: string;
    impact: string;
    reason: "below_severity_threshold" | "not_in_allowlist";
  }>;
  diff?: {
    original: string;
    patched: string;
  };
}
```

---

## `audit_component`

Raw axe-core audit. Returns violations without scoring or suggestions.

**Input:**
```ts
{ path: string } // URL or absolute file path
```

**Output:**
```ts
{
  url: string;
  violations: Violation[];
  passes: number;
  incomplete: number;
  timestamp: string;
}
```

**Violation:**
```ts
{
  id: string;
  impact: "minor" | "moderate" | "serious" | "critical";
  description: string;
  help: string;
  helpUrl: string;
  wcag: string[];
  nodes: Array<{
    selector: string;
    html: string;
    failureSummary: string;
  }>;
}
```

---

## `fix_component`

Patches a single violation in a source file.

**Input:**
```ts
{
  path: string;           // absolute path to .tsx / .jsx / .html
  violation_id: string;   // axe rule ID
  write?: boolean;        // write to disk (default: false)
}
```

**Supported violation IDs:**

| ID | Patches | WCAG |
|---|---|---|
| `image-alt` | Adds `alt=""` to img elements (`.tsx`/`.jsx`/`.html`/`.vue`/`.svelte`) | 1.1.1 (A) |
| `button-name` | Adds `aria-label` to nameless buttons | 4.1.2 (A) |
| `link-name` | Adds `aria-label` to nameless links | 2.4.4 (A) |
| `label` | Annotates inputs missing a label | 1.3.1, 4.1.2 (A) |
| `aria-label` | Annotates interactive elements missing accessible names | 4.1.2 (A) |
| `html-has-lang` | Adds `lang="en"` to html element | 3.1.1 (A) |
| `color-contrast` | Returns explanation only — cannot auto-patch | 1.4.3 (AA) |
| `heading-order` | Returns explanation only — cannot auto-patch | 1.3.1 (A) |

**Output:**
```ts
{
  violation_id: string;
  original: string;
  patched: string;
  explanation: string;
  wcag: string;
  changed: boolean;
}
```

---

## `crawl_site`

Crawls a website from a start URL or sitemap, audits multiple same-origin pages, and aggregates findings.

**Input:**
```ts
{
  start_url?: string;        // crawl from this URL, discovering same-origin links
  sitemap_url?: string;      // seed from a sitemap.xml
  max_pages?: number;        // default: 10, max: 50
  include_html_snippets?: boolean;
}
```
One of `start_url` or `sitemap_url` is required.

**Output:**
```ts
{
  start_url: string;
  pages_scanned: number;
  aggregate_score: number;          // average across pages
  worst_pages: Array<{ url: string; score: number; grade: string }>;
  most_common_violations: Array<{ id: string; count: number; impact: string }>;
  pages: Array<{
    url: string;
    score: number;
    grade: string;
    wcag_level: string;
    violation_count: number;
    critical_count: number;
  }>;
  timestamp: string;
}
```

---

## `audit_repo`

Scans a project directory and returns violations across all files, sorted by severity.

**Input:**
```ts
{
  root: string;          // absolute path to project root
  baseUrl?: string;      // live dev server — required for TSX/JSX files
  maxFiles?: number;     // default: 20, max: 100
}
```

**Output:**
```ts
{
  root: string;
  filesScanned: number;
  totalViolations: number;
  criticalViolations: number;
  topViolations: Array<{ id: string; count: number; impact: string }>;
  fileResults: Array<{
    file: string;
    violationCount: number;
    criticalCount: number;
    seriousCount: number;
    violations: Array<{
      id: string;
      impact: string;
      description: string;
      wcag: string[];
      nodeCount: number;
    }>;
  }>;
  timestamp: string;
}
```

---

## CLI commands

Same engine as MCP. Use when shell is available.

### `loop11y audit <url>`

Audit a single live URL.

Flags: `--json` (default) · `--markdown` · `--sarif` · `--html` · `--output <file>` · `--storage-state <path>` · `--header 'k: v'` (repeatable) · `--basic-auth-user <u>` · `--basic-auth-pass <p>` · `--fail-on critical|serious|moderate|minor` · `--max-violations <n>` · `--baseline <report.json>`.

### `loop11y audit:file <path>`

Audit a local `.html` file. Same output flags as `audit`.

### `loop11y audit:repo <path>`

Scan a checked-out project. Flags: `--max-files <n>` (default 20, max 100) · `--base-url <url>` (required for `.tsx`/`.jsx`/`.vue`/`.svelte`).

### `loop11y crawl`

One of `--url <start>`, `--sitemap <url>`, or `--routes <file>` required. Flags: `--max-pages <n>` (default 10, max 50) · `--include-pattern <regex>` · `--exclude-pattern <regex>` · output + threshold flags as above.

Auto-detects `/sitemap.xml`, `/sitemap_index.xml`, `robots.txt` `Sitemap:` directives. Tracking params (`utm_*`, `gclid`, `fbclid`) stripped pre-dedupe. SPA routes discovered post-hydration.

### `loop11y verify <source-path> --url <url>`

Re-audit URL after a remediation. Returns score and delta vs prior audit if `--baseline` given. **CLI / HTTP only — not MCP.** From MCP, just call `evaluate` again.

### `loop11y diff <before.json> <after.json>`

Build a before/after HTML or JSON report. Flags: `--output <file>` · `--html` · `--json`.

---

## HTTP API

Start: `LOOP11Y_PORT=3000 npx -y loop11y` (or Docker / Fly).

| Endpoint | Body shape | Notes |
|---|---|---|
| `POST /api/evaluate` | `{ url, include_html_snippets?, include_passing? }` | Same as MCP `evaluate`. |
| `POST /api/crawl` | `{ start_url?, sitemap_url?, max_pages?, include_html_snippets? }` | Same as MCP `crawl_site`. |
| `POST /api/repo-audit` | `{ root, baseUrl?, maxFiles? }` | Same as MCP `audit_repo`. |
| `POST /api/remediate` | `{ source_path, audit_url, mode, min_severity?, only? }` | Same as MCP `remediate`. |
| `POST /api/verify` | `{ source_path, url, baseline? }` | CLI / HTTP only. |
| `POST /mcp` | streamable HTTP MCP transport | For MCP clients that prefer HTTP over stdio. |
| `GET /openapi.json` | — | Full OpenAPI 3 spec. Wire into ChatGPT custom GPT Action. |
| `GET /.well-known/ai-plugin.json` | — | Plugin manifest. |
| `GET /health` | — | Liveness. |

---

## GitHub Action

```yaml
- uses: tayyabataimur/loop11y/action@v0.1.0
  with:
    url: https://staging.example.com   # or path: ./ for repo audit
    fail-under: 90                     # fail job below this score
    fail-on: critical                  # alt gate: fail on any of this severity
    output: report.md                  # written + posted as PR comment
```

Composite Action — runs the CLI internally. Posts Markdown report as PR comment, sets check status.

---

## Harness SDK

```ts
import { Loop11yClient } from "loop11y/harness-sdk";

const client = new Loop11yClient({ baseUrl: "http://localhost:3000" });
const audit = await client.evaluate({ url: "https://example.com" });
const fix   = await client.remediate({ source_path: "...", audit_url: "...", mode: "diff" });
```

Lightweight Node client over the HTTP API. Use when building custom harnesses, agents, or scripts.

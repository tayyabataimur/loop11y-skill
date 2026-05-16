# Loop11y Tool Reference

Complete input/output documentation for all six tools.

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
| `image-alt` | Adds `alt=""` to img elements | 1.1.1 (A) |
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

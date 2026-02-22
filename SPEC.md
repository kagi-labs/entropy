# Metsuke 目付

**The Code Inspector. Tracks code entropy, captures review mistakes as enforceable rules, and scans dependencies for vulnerabilities.**

Inspired by HashiCorp-style tooling: single binary, focused purpose, zero dependencies. Named after the Edo-period inspectors (目付) who monitored conduct and enforced discipline across the shogunate.

## Vision

AI-accelerated development creates technical debt faster than teams realize. Mistakes get repeated because lessons aren't captured. Code reviews find the same issues over and over.

**Metsuke** closes the loop: plan → develop → review → capture mistakes → enforce as rules → better future code. Every mistake becomes a permanent guardrail.

## The Feedback Loop

```
    ┌──────────┐
    │   Plan   │ ◄── Rules inform future plans
    └────┬─────┘
         ▼
    ┌──────────┐
    │ Develop  │ ◄── Rules enforced during dev
    └────┬─────┘
         ▼
    ┌──────────┐
    │  Review  │ ── Mistakes found here...
    └────┬─────┘
         ▼
    ┌──────────┐
    │  Track   │ ── ...get recorded as data...
    └────┬─────┘
         ▼
    ┌──────────┐
    │  Rules   │ ── ...and become enforceable rules
    └──────────┘
```

## CLI Commands

```bash
metsuke init               # Initialize .metsuke/ in a project
metsuke score              # Calculate current health score
metsuke score --json       # Machine-readable output
metsuke check              # Run all rules against staged/changed files
metsuke check --ci         # Strict mode for CI (non-zero exit on violations)
metsuke deps               # Scan dependencies for known vulnerabilities
metsuke deps --severity high  # Filter by severity
metsuke add-rule           # Capture a new rule from a mistake
metsuke trend              # Show score history (ASCII chart in terminal)
metsuke diff [branch]      # Compare entropy between current and target branch
metsuke blame [file]       # Show which changes introduced the most entropy
metsuke scan --ai          # Deep AI-powered security & complexity analysis
metsuke doctor             # Diagnose codebase health issues with suggestions
metsuke export             # Export rules + scores for external dashboards
metsuke serve              # Local web dashboard (localhost)
metsuke mcp                # Start MCP server for agent integration

# Codebase Index & Navigation
metsuke index              # Build/update codebase index (incremental)
metsuke symbols [query]    # Search symbols (functions, types, interfaces)
metsuke callers [func]     # Show what calls this function
metsuke deps [pkg]         # Show package dependency graph
metsuke map [path]         # Show module structure overview
```

## Core Components

### 1. Entropy Metrics (Scanner)

Measures codebase health across multiple dimensions:

| Metric | What it measures | How it's calculated | Weight |
|--------|-----------------|---------------------|--------|
| **Cyclomatic complexity** | Control flow complexity per function | Count decision points: `if`, `for`, `switch`, `case`, `&&`, `\|\|`. Score = branches + 1. Uses `go/ast` to walk AST and count `*ast.IfStmt`, `*ast.ForStmt`, `*ast.SwitchStmt`, `*ast.BinaryExpr` | 20% |
| **Duplication ratio** | Copy-paste / near-duplicate code blocks | Hash rolling windows of N lines (default 6). Normalize whitespace, MD5 each window, find collisions across files. Similar to PMD/CPD algorithm | 15% |
| **Coupling score** | Cross-module/package dependencies | Parse imports per package, build dependency graph. Measure fan-in (who depends on me) and fan-out (who do I depend on). High fan-out = high coupling | 15% |
| **File churn** | Files changing too frequently (design smell) | `git log --numstat` over last N commits. Count changes per file, weighted by recency (recent changes count more). High churn relative to file size = design smell | 10% |
| **Convention drift** | Inconsistent patterns across codebase | Detect naming style inconsistencies (camelCase vs snake_case mix), error handling pattern variance, file structure anomalies | 10% |
| **Dead code** | Unreachable or unused exports | Exported symbols never referenced outside their package. `go/ast` for declarations + cross-reference analysis | 10% |
| **Dependency vulnerabilities** | Known CVEs in dependencies | Parse `go.mod`/`package.json`/`requirements.txt`/`Cargo.toml`, query [OSV API](https://osv.dev) for known vulnerabilities. Count by severity (critical/high/medium/low) | 20% |

#### Score Calculation

```
score = 100 - Σ penalties

Where each penalty is normalized to its weight:
  complexity_penalty  = min(weight, (avg_complexity / threshold) * weight)
  duplication_penalty = min(weight, (dup_ratio / threshold) * weight)
  coupling_penalty    = min(weight, (avg_fanout / threshold) * weight)
  churn_penalty       = min(weight, (churn_ratio / threshold) * weight)
  drift_penalty       = min(weight, (drift_score / threshold) * weight)
  dead_code_penalty   = min(weight, (dead_ratio / threshold) * weight)
  vuln_penalty        = min(weight, (weighted_vuln_count / threshold) * weight)
```

**Default thresholds** (configurable in `config.yaml`):

| Metric | Green | Yellow | Red |
|--------|-------|--------|-----|
| Complexity per function | < 10 | 10-20 | > 20 |
| Duplication ratio | < 3% | 3-8% | > 8% |
| Avg fan-out per package | < 5 | 5-10 | > 10 |
| File churn (changes/week) | < 5 | 5-15 | > 15 |
| Dead code ratio | < 2% | 2-5% | > 5% |
| Critical/High CVEs | 0 | 1-2 | > 2 |

Output: **Health score (0-100)** per file, per module, and aggregate.

#### Language Support

| Language | Parser | Dependency file | Status |
|----------|--------|----------------|--------|
| Go | `go/ast` (stdlib) | `go.mod` | Primary |
| Python | tree-sitter | `requirements.txt`, `pyproject.toml` | Planned |
| TypeScript/JS | tree-sitter | `package.json` | Planned |
| Rust | tree-sitter | `Cargo.toml` | Planned |

### 1b. Dependency Security Scanner

Checks project dependencies against the [OSV (Open Source Vulnerabilities)](https://osv.dev) database — Google's free, open, universal vulnerability database covering Go, npm, PyPI, crates.io, and more.

```bash
metsuke deps                 # Scan dependencies, check for known vulns
metsuke deps --format json   # Machine-readable output for CI
metsuke deps --severity high # Only show high/critical
```

**How it works:**
1. Parse dependency manifest (`go.mod`, `package.json`, etc.)
2. Extract package names + versions
3. Query OSV API: `POST https://api.osv.dev/v1/query`
4. Report findings grouped by severity

**Example output:**
```
Dependency Security Report
──────────────────────────
📦 3 dependencies scanned, 2 vulnerabilities found

CRITICAL  golang.org/x/crypto@v0.17.0
          GHSA-45x7-px36-x8w8 — SSH server auth bypass
          Fixed in: v0.31.0
          → go get golang.org/x/crypto@latest

MEDIUM    golang.org/x/net@v0.19.0
          CVE-2023-45288 — HTTP/2 rapid reset
          Fixed in: v0.23.0
          → go get golang.org/x/net@latest

Score impact: -15 (2 vulns: 1 critical, 1 medium)
```

**Language-specific tools metsuke wraps:**
- Go: `govulncheck` (official, reachability analysis)
- Python: `pip-audit`
- Node: `npm audit`
- Rust: `cargo audit`

Falls back to OSV API when language-specific tools aren't installed.

### 2. Rule System (Mistake → Knowledge → Enforcement)

When a mistake is found during review, capture it:

```yaml
# .metsuke/rules/no-raw-sql.yaml
id: no-raw-sql
severity: error
category: security
learned_from: "PR #42 — SQL injection in user search"
date: 2026-02-22
description: "Never use raw SQL string concatenation. Always use parameterized queries."
pattern: "fmt.Sprintf.*SELECT.*%s"
fix: "Use db.Query with $1 placeholders"
languages: [go]
```

**Rule properties:**
- **Project-scoped** — stored in `.metsuke/rules/`, committed with code
- **Machine-readable** — YAML with regex patterns
- **Traceable** — linked back to the PR/review where the mistake was found
- **Enforceable** — checked in CI, pre-commit, or editor

**Rule templates** (built-in packs):
- `metsuke init --pack security` — OWASP top 10 patterns
- `metsuke init --pack performance` — common perf anti-patterns
- `metsuke init --pack ai-hygiene` — AI-generated code smells

### 3. Storage

#### Local (default — free, self-contained)

**SQLite** for time-series data (scores, events, trends). YAML files for human-editable rules.

```
.metsuke/
├── metsuke.db              # SQLite: scores, events, trends
├── config.yaml             # Project-level settings
└── rules/                  # YAML rule files (human-editable, git-trackable)
    ├── no-raw-sql.yaml
    ├── consistent-errors.yaml
    └── _packs/             # Built-in rule packs
        └── security.yaml
```

**Why SQLite + YAML:**
- SQLite: fast queries for trending, dashboards, CI reporting. No server needed.
- YAML rules: human-readable, diffable in PRs, easy to write by hand or generate.
- Everything in `.metsuke/` — git-trackable, portable, zero external dependencies.

#### Export (self-hosted teams)

For teams that want to aggregate data across repos into their own infrastructure:

```bash
metsuke export --format json           # Dump scores + rules as JSON
metsuke export --db postgres://...     # Push to external PostgreSQL/MySQL
metsuke export --s3 s3://bucket/path   # Push to S3-compatible storage
metsuke export --webhook https://...   # POST to any endpoint (CI integration)
```

Teams can pipe this into Grafana, Datadog, their own BI tools — whatever they already use.

#### Metsuke Cloud (hosted SaaS — paid tier)

**The business model:** Open-source CLI is free forever. Hosted dashboard is subscription-based.

Teams connect their repos → data flows to our hosted platform → we provide:

```
┌─────────────────────────────────────────────────────┐
│                 Metsuke Cloud                        │
├─────────────────────────────────────────────────────┤
│                                                      │
│  📊 Team Dashboard                                   │
│  ├── Score trends across all repos                   │
│  ├── Cross-repo rule violation heatmap               │
│  ├── Per-developer entropy contribution              │
│  └── Historical comparison (this sprint vs last)     │
│                                                      │
│  🔔 Notifications & Alerts                           │
│  ├── Slack/Discord/Email when score drops             │
│  ├── Weekly digest: "Top 5 entropy sources"          │
│  └── PR-level: "This PR adds +8 entropy"             │
│                                                      │
│  📈 Analytics & Reporting                            │
│  ├── Team velocity vs entropy correlation            │
│  ├── "Entropy budget" per sprint                     │
│  ├── Dependency risk across all projects             │
│  └── Executive summaries (PDF export)                │
│                                                      │
│  🤝 Team Collaboration                               │
│  ├── Shared rule libraries across org                │
│  ├── Review assignments based on entropy ownership   │
│  └── Onboarding: "Here's what we've learned"        │
│                                                      │
└─────────────────────────────────────────────────────┘
```

**How data flows:**
```
Developer machine / CI                    Metsuke Cloud
┌───────────────┐                        ┌──────────────┐
│ metsuke check │── scores + rules ───→  │  Ingest API  │
│ metsuke score │   (JSON over HTTPS)    │              │
│ metsuke deps  │                        │  PostgreSQL  │
└───────────────┘                        │  + S3        │
                                         │              │
       ◄──── dashboards, alerts, ────────│  Web UI      │
             notifications               └──────────────┘
```

```bash
# Connect a repo to Metsuke Cloud
metsuke cloud login
metsuke cloud connect          # Links current repo to your org
metsuke cloud push             # Manual push (or auto via CI)
metsuke cloud status           # Check sync status
```

**Pricing model (planned):**

| Tier | Price | Includes |
|------|-------|---------|
| **Open Source** | Free forever | CLI, local SQLite, all metrics, rules, local dashboard |
| **Team** | $/month per repo | Hosted dashboard, alerts, cross-repo analytics, shared rules |
| **Enterprise** | Custom | SSO, audit logs, SLA, custom integrations, on-prem option |

### 4. Codebase Index & Navigation

Metsuke already parses the entire codebase for entropy scoring. Instead of discarding that structural data, we persist it as a queryable index — turning Metsuke into a **codebase GPS for AI agents**.

**What gets indexed:**
- **File tree** — module/package structure
- **Symbols** — all functions, types, interfaces, constants, variables with locations
- **Call graph** — who calls who (static analysis)
- **Import graph** — package-level dependencies
- **Doc comments** — godoc, JSDoc, docstrings
- **Git ownership** — blame data (who owns what, last modified)

**Storage:** Same SQLite database (`metsuke.db`). Index updates incrementally on file change (MD5 hash check, like the entropy scanner).

**CLI usage:**
```bash
metsuke index                          # Build/update index
metsuke symbols User                   # Find all symbols matching "User"
metsuke symbols --type func auth       # Only functions matching "auth"
metsuke callers validateToken          # What calls validateToken()?
metsuke deps pkg/api                   # Dependency graph for a package
metsuke map .                          # Overview of module structure
metsuke map --depth 2                  # Limit depth
```

**Example output:**
```
$ metsuke symbols --type func auth

  pkg/auth/handler.go:42      func HandleLogin(w, r)
  pkg/auth/handler.go:87      func HandleLogout(w, r)
  pkg/auth/middleware.go:15    func RequireAuth(next) Handler
  pkg/auth/token.go:23        func ValidateToken(token) (*Claims, error)
  pkg/auth/token.go:56        func RefreshToken(old) (string, error)

5 symbols found

$ metsuke callers ValidateToken

  pkg/auth/middleware.go:22    RequireAuth()
  pkg/api/routes.go:45        handleProtectedRoute()
  pkg/ws/upgrade.go:18        upgradeWebSocket()

3 callers found

$ metsuke map --depth 2

  pkg/
  ├── api/          (12 files, 8 exported funcs, fan-out: 4)
  ├── auth/         (5 files, 6 exported funcs, fan-out: 2)
  ├── db/           (7 files, 11 exported funcs, fan-out: 1)
  ├── models/       (4 files, 9 exported types, fan-out: 0)
  └── ws/           (3 files, 4 exported funcs, fan-out: 3)
```

**Why this belongs in Metsuke (not a separate tool):**
- The scanner already walks the full AST — indexing is a near-zero-cost addition
- Same SQLite database, same incremental update logic
- Coupling/dependency metrics need the import graph anyway
- One binary, one MCP server, one `metsuke.db` — agents get health + navigation in one tool

### 5. Dashboard

**Terminal UI** (`metsuke trend`):
```
Health Score — last 30 days
100 ┤
 90 ┤          ╭──╮
 80 ┤    ╭─────╯  ╰───╮
 70 ┤────╯             ╰──╮
 60 ┤                     ╰────
 50 ┤
    └────────────────────────────
     Jan 23        Feb 06       Feb 22

Top issues:
  ⚠ auth/handler.go    complexity: 42 (limit: 20)
  ⚠ api/routes.go      duplication: 3 blocks
  ✗ db/queries.go      rule: no-raw-sql
```

**Web dashboard** (`metsuke serve`):
- Local-only web UI on localhost
- Score trends over time (charts)
- Rule violation heatmap
- File-level drill-down
- Exportable reports (HTML, JSON)

### 6. CI/CD Integration

**GitHub Actions:**
```yaml
- name: Metsuke Check
  uses: kagi-labs/metsuke-action@v1
  with:
    fail-on: error          # Fail build on error-severity rules
    score-threshold: 70     # Fail if health drops below 70
    comment: true           # Post score as PR comment
```

**Pre-commit hook:**
```yaml
# .pre-commit-config.yaml
- repo: https://github.com/kagi-labs/metsuke
  hooks:
    - id: metsuke-check
```

**GitLab CI, Jenkins, etc.:** Just run `metsuke check --ci`

### 7. AI Security & Complexity Analysis

Inspired by [Anthropic's Claude Code Security](https://www.anthropic.com/news/claude-code-security) — but open, local-first, and integrated with the kagi-labs agent OS.

**AI-powered analysis** (beyond regex pattern matching):
- **Semantic security scan:** Use LLM reasoning to find business logic flaws, broken access control, data flow issues — not just pattern matches
- **Multi-stage verification:** Find an issue → attempt to disprove it → only report confirmed findings (reduces false positives)
- **Confidence ratings:** Each finding gets a confidence score (high: deterministic check, medium: heuristic, low: AI inference)
- **AI anti-pattern detection:** Detect AI-generated code smells (excessive abstraction, inconsistent naming, duplicated-with-slight-variation patterns)
- **Auto-rule suggestion:** After N similar review comments, suggest creating a rule
- **CLAUDE.md / .cursorrules sync:** Export metsuke rules as AI coding assistant instructions

```bash
metsuke scan --ai          # Deep AI-powered analysis (requires LLM)
metsuke scan --ai --model local  # Use local model (Ollama)
metsuke scan --ai --model api    # Use API (Claude/GPT)
```

### 8. Kagi Labs Ecosystem Integration

Metsuke is standalone but designed to plug into the kagi-labs agent OS:

```
┌─────────────────────────────────────────────────────────┐
│                    Agent OS Layer                        │
├──────────┬───────────┬──────────┬───────────────────────┤
│  Mikado  │  Minato   │  Hashi   │  Aegis                │
│  (soul)  │  (comms)  │  (tasks) │  (security)           │
├──────────┴───────────┴──────────┴───────────────────────┤
│                   Tool Layer                             │
├──────────┬───────────┬──────────────────────────────────┤
│ Metsuke  │  Kura     │  Kaji                             │
│ (inspect)│  (store)  │  (forge)                          │
└──────────┴───────────┴──────────────────────────────────┘
```

**Integration points:**

| System | How Metsuke connects | Direction |
|--------|---------------------|-----------|
| **Aegis** (security control plane) | Metsuke feeds security findings to Aegis for human-in-the-loop approval before auto-fixes. Aegis policy rules can trigger metsuke scans on tool calls | Metsuke ↔ Aegis |
| **Hashi** (task engine) | Hashi delegates `metsuke check` as a task in CI/review workflows. Metsuke results inform Hashi's task planning (skip deploy if score < threshold) | Hashi → Metsuke |
| **Kaji** (agent forge) | Kaji's review phase runs `metsuke check` automatically. Findings feed back into spec refinement. Codex reviewer gets metsuke context | Kaji → Metsuke |
| **Kura** (storehouse) | Metsuke scores + rule violation history stored in Kura for cross-project search and long-term trending | Metsuke → Kura |
| **Mikado** (soul/nervous system) | Metsuke health events published to Mikado's event bus. Score drops trigger alerts through the nervous system | Metsuke → Mikado |
| **Minato** (channel harbor) | Metsuke reports routed through Minato to Discord/Slack. "Score dropped to 62 on project X" | Metsuke → Minato |

**MCP server mode:**
```bash
metsuke mcp                # Start MCP server — expose rules and scores to any AI agent
```

Tools exposed via MCP:

**Health & Rules:**
- `metsuke_score(path)` — get health score for a codebase
- `metsuke_check(path, files)` — check specific files against rules
- `metsuke_rules(path)` — list all project rules
- `metsuke_deps(path)` — check dependency vulnerabilities
- `metsuke_add_rule(rule)` — create a new rule programmatically

**Codebase Navigation:**
- `metsuke_symbols(query, type?)` — search symbols (functions, types, interfaces)
- `metsuke_callers(func)` — what calls this function?
- `metsuke_map(path, depth?)` — module/package structure overview
- `metsuke_find(description)` — natural language: "files that handle authentication"
- `metsuke_deps_graph(pkg)` — package dependency graph

This means any agent in the ecosystem (Claude, Codex, local models) can query health data AND navigate the codebase through a single MCP server.

### 9. Team Features (Future)

- **Shared rule registry:** Publish/subscribe to rule packs (`metsuke registry push/pull`)
- **Org-wide baselines:** Set minimum scores across repos
- **Review integration:** Auto-tag PRs with entropy impact (`+3 entropy`, `-5 entropy`)

## Architecture

```
┌───────────────────────────────────────────────────────────┐
│                      CLI (cobra)                          │
├──────┬──────┬──────┬──────┬───────┬──────┬───────────────┤
│score │check │deps  │scan  │trend  │serve │ cloud / mcp   │
├──────┴──────┴──────┴──────┴───────┴──────┴───────────────┤
│                     Core Engine                           │
├──────────┬──────────┬────────────┬───────────┬───────────┤
│ Scanner  │Rule Eval │Dep Auditor │ AI Engine │ Indexer   │
├──────────┴──────────┴────────────┴───────────┴───────────┤
│                    Storage Layer                          │
├─────────────┬──────────────┬──────────────┬──────────────┤
│ SQLite      │ YAML (rules) │ OSV API      │ Export       │
│ (local)     │              │ (deps)       │ (pg/s3/hook) │
├─────────────┴──────────────┴──────────────┴──────────────┤
│                  Integration Layer                        │
├──────────┬─────────┬──────────┬──────────┬───────────────┤
│ MCP      │ Aegis   │ Hashi    │ Minato   │ Cloud API     │
│ (agents) │ (sec)   │ (tasks)  │ (comms)  │ (SaaS)        │
└──────────┴─────────┴──────────┴──────────┴───────────────┘
```

**Tech stack:**
- **Language:** Go
- **CLI framework:** Cobra
- **DB:** SQLite (via modernc.org/sqlite — pure Go, no CGO)
- **Parsing:** go/ast (Go), tree-sitter (multi-lang)
- **AI engine:** Ollama (local) or Claude/GPT API (remote) — optional, core metrics work without AI
- **Dashboard:** Terminal charts (termdash or lipgloss), embedded web UI (templ + htmx)
- **Integration:** MCP server, event publishing (Mikado-compatible)

## MVP Scope

**Phase 1: Foundation**
- [ ] `metsuke init` — create `.metsuke/` structure
- [ ] `metsuke score` — calculate complexity + duplication for Go codebases
- [ ] JSON + human-readable output
- [ ] SQLite storage for score history

**Phase 2: Rules**
- [ ] YAML rule format
- [ ] `metsuke check` — match rules against files
- [ ] `metsuke add-rule` — capture new rule
- [ ] Pre-commit hook support

**Phase 3: Trends & Dashboard**
- [ ] `metsuke trend` — ASCII chart in terminal
- [ ] `metsuke serve` — local web dashboard
- [ ] `metsuke diff` — compare branches
- [ ] Score threshold alerts

**Phase 4: AI Security & Complexity Analysis**
- [ ] `metsuke scan --ai` — LLM-powered semantic analysis
- [ ] Multi-stage verification (find → disprove → confirm)
- [ ] Confidence ratings on findings
- [ ] Local model support (Ollama)
- [ ] CLAUDE.md / .cursorrules export

**Phase 5: Export & Ecosystem Integration**
- [ ] `metsuke export` — JSON, PostgreSQL, S3, webhook targets
- [ ] MCP server mode (`metsuke mcp`)
- [ ] Aegis integration (security control plane)
- [ ] Hashi integration (task delegation)
- [ ] Minato integration (alert routing)
- [ ] Kura integration (cross-project storage)

**Phase 6: Metsuke Cloud (SaaS)**
- [ ] Ingest API — receive scores/rules from CLI over HTTPS
- [ ] Hosted PostgreSQL + S3 backend
- [ ] Team dashboard — cross-repo trends, heatmaps, per-dev stats
- [ ] Notifications — Slack/Discord/email alerts on score drops
- [ ] Analytics — velocity vs entropy correlation, sprint budgets
- [ ] `metsuke cloud login/connect/push/status` CLI commands

**Phase 7: Multi-lang & Team**
- [ ] Python + TypeScript support via tree-sitter
- [ ] Rule packs (security, performance, ai-hygiene)
- [ ] Shared registry (publish/pull rule packs)
- [ ] GitHub Action
- [ ] PR entropy impact comments

## Why This Matters

> "The best time to fix a bug is when you first learn about it. The second best time is to make sure it never happens again."

Most teams find the same mistakes repeatedly. Metsuke closes the loop: **every mistake becomes a permanent guardrail.**

# Entropy

**A standalone CLI tool for tracking code entropy and turning review mistakes into enforceable rules.**

Inspired by HashiCorp-style tooling: single binary, focused purpose, zero dependencies.

## Vision

AI-accelerated development creates technical debt faster than teams realize. Mistakes get repeated because lessons aren't captured. Code reviews find the same issues over and over.

**Entropy** closes the loop: plan → develop → review → capture mistakes → enforce as rules → better future code. Every mistake becomes a permanent guardrail.

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
entropy init               # Initialize .entropy/ in a project
entropy score              # Calculate current health score
entropy score --json       # Machine-readable output
entropy check              # Run all rules against staged/changed files
entropy check --ci         # Strict mode for CI (non-zero exit on violations)
entropy deps               # Scan dependencies for known vulnerabilities
entropy deps --severity high  # Filter by severity
entropy add-rule           # Capture a new rule from a mistake
entropy trend              # Show score history (ASCII chart in terminal)
entropy diff [branch]      # Compare entropy between current and target branch
entropy blame [file]       # Show which changes introduced the most entropy
entropy doctor             # Diagnose codebase health issues with suggestions
entropy export             # Export rules + scores for external dashboards
entropy serve              # Local web dashboard (localhost)
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
entropy deps                 # Scan dependencies, check for known vulns
entropy deps --format json   # Machine-readable output for CI
entropy deps --severity high # Only show high/critical
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

**Language-specific tools entropy wraps:**
- Go: `govulncheck` (official, reachability analysis)
- Python: `pip-audit`
- Node: `npm audit`
- Rust: `cargo audit`

Falls back to OSV API when language-specific tools aren't installed.

### 2. Rule System (Mistake → Knowledge → Enforcement)

When a mistake is found during review, capture it:

```yaml
# .entropy/rules/no-raw-sql.yaml
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
- **Project-scoped** — stored in `.entropy/rules/`, committed with code
- **Machine-readable** — YAML with regex patterns
- **Traceable** — linked back to the PR/review where the mistake was found
- **Enforceable** — checked in CI, pre-commit, or editor

**Rule templates** (built-in packs):
- `entropy init --pack security` — OWASP top 10 patterns
- `entropy init --pack performance` — common perf anti-patterns
- `entropy init --pack ai-hygiene` — AI-generated code smells

### 3. Storage

**SQLite** for time-series data (scores, events, trends). YAML files for human-editable rules. Best of both worlds.

```
.entropy/
├── entropy.db              # SQLite: scores, events, trends
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
- Everything in `.entropy/` — git-trackable, portable, zero external dependencies.

### 4. Dashboard

**Terminal UI** (`entropy trend`):
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

**Web dashboard** (`entropy serve`):
- Local-only web UI on localhost
- Score trends over time (charts)
- Rule violation heatmap
- File-level drill-down
- Exportable reports (HTML, JSON)

### 5. CI/CD Integration

**GitHub Actions:**
```yaml
- name: Entropy Check
  uses: kagi-labs/entropy-action@v1
  with:
    fail-on: error          # Fail build on error-severity rules
    score-threshold: 70     # Fail if health drops below 70
    comment: true           # Post score as PR comment
```

**Pre-commit hook:**
```yaml
# .pre-commit-config.yaml
- repo: https://github.com/kagi-labs/entropy
  hooks:
    - id: entropy-check
```

**GitLab CI, Jenkins, etc.:** Just run `entropy check --ci`

### 6. AI-Aware Features

- **Anti-pattern detection:** Detect common AI-generated code smells (excessive abstraction, inconsistent naming, duplicated-with-slight-variation patterns)
- **Auto-rule suggestion:** After N similar review comments, suggest creating a rule
- **CLAUDE.md / .cursorrules sync:** Export entropy rules as AI coding assistant instructions
- **MCP server mode:** `entropy mcp` — expose rules and scores to AI agents via Model Context Protocol

### 7. Team Features (Future)

- **Shared rule registry:** Publish/subscribe to rule packs (`entropy registry push/pull`)
- **Org-wide baselines:** Set minimum scores across repos
- **Review integration:** Auto-tag PRs with entropy impact (`+3 entropy`, `-5 entropy`)

## Architecture

```
┌──────────────────────────────────────────────────┐
│                  CLI (cobra)                      │
├──────┬──────┬──────┬───────┬──────┬──────────────┤
│score │check │deps  │trend  │serve │ add-rule     │
├──────┴──────┴──────┴───────┴──────┴──────────────┤
│                 Core Engine                       │
├──────────┬──────────┬────────────┬───────────────┤
│ Scanner  │Rule Eval │Dep Auditor │ Trend Engine  │
├──────────┴──────────┴────────────┴───────────────┤
│                Storage Layer                      │
├─────────────┬──────────────┬─────────────────────┤
│ SQLite (db) │ YAML (rules) │ OSV API (deps)      │
└─────────────┴──────────────┴─────────────────────┘
```

**Tech stack:**
- **Language:** Go
- **CLI framework:** Cobra
- **DB:** SQLite (via modernc.org/sqlite — pure Go, no CGO)
- **Parsing:** go/ast (Go), tree-sitter (multi-lang)
- **Dashboard:** Terminal charts (termdash or lipgloss), embedded web UI (templ + htmx)

## MVP Scope

**Phase 1: Foundation**
- [ ] `entropy init` — create `.entropy/` structure
- [ ] `entropy score` — calculate complexity + duplication for Go codebases
- [ ] JSON + human-readable output
- [ ] SQLite storage for score history

**Phase 2: Rules**
- [ ] YAML rule format
- [ ] `entropy check` — match rules against files
- [ ] `entropy add-rule` — capture new rule
- [ ] Pre-commit hook support

**Phase 3: Trends & Dashboard**
- [ ] `entropy trend` — ASCII chart in terminal
- [ ] `entropy serve` — local web dashboard
- [ ] `entropy diff` — compare branches
- [ ] Score threshold alerts

**Phase 4: AI & Multi-lang**
- [ ] AI anti-pattern detection
- [ ] CLAUDE.md / .cursorrules export
- [ ] Python + TypeScript support via tree-sitter
- [ ] MCP server mode

**Phase 5: Team & Registry**
- [ ] Rule packs (security, performance, ai-hygiene)
- [ ] Shared registry (publish/pull rule packs)
- [ ] GitHub Action
- [ ] PR entropy impact comments

## Why This Matters

> "The best time to fix a bug is when you first learn about it. The second best time is to make sure it never happens again."

Most teams find the same mistakes repeatedly. Entropy closes the loop: **every mistake becomes a permanent guardrail.**

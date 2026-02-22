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

| Metric | What it measures | Weight |
|--------|-----------------|--------|
| **Cyclomatic complexity** | Control flow complexity per function | 25% |
| **Duplication ratio** | Copy-paste / near-duplicate code blocks | 20% |
| **Coupling score** | Cross-module/package dependencies | 15% |
| **File churn** | Files changing too frequently (design smell) | 15% |
| **Convention drift** | Inconsistent patterns across codebase | 15% |
| **Dead code** | Unreachable or unused exports | 10% |

Output: **Health score (0-100)** per file, per module, and aggregate.

Language support (planned):
- Go (primary)
- Python
- TypeScript/JavaScript
- Rust

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
  uses: olesbsn/entropy-action@v1
  with:
    fail-on: error          # Fail build on error-severity rules
    score-threshold: 70     # Fail if health drops below 70
    comment: true           # Post score as PR comment
```

**Pre-commit hook:**
```yaml
# .pre-commit-config.yaml
- repo: https://github.com/olesbsn/entropy
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
┌─────────────────────────────────────────┐
│              CLI (cobra)                │
├──────┬──────┬───────┬──────┬────────────┤
│score │check │trend  │serve │ add-rule   │
├──────┴──────┴───────┴──────┴────────────┤
│            Core Engine                  │
├────────┬──────────┬─────────────────────┤
│Scanner │Rule Eval │ Trend Engine        │
├────────┴──────────┴─────────────────────┤
│         Storage Layer                   │
├──────────────┬──────────────────────────┤
│ SQLite (db)  │ YAML (rules)            │
└──────────────┴──────────────────────────┘
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

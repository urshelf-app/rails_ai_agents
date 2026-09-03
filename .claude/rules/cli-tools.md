# CLI Tools

Prefer these tools over their standard equivalents in all shell commands.

## Code search and navigation

**Query the knowledge graph before reading or grepping source.** When `graphify-out/graph.json`
exists, the graph is the primary way to understand this codebase. It returns a scoped subgraph —
usually far smaller than the output of a recursive grep or a full file read — and it follows
relationships (callers, callees, cross-file references, dynamic dispatch) that text search cannot.

| Need | Command |
|---|---|
| Understand an area, answer a codebase question | `graphify query "<question>"` |
| How two things relate | `graphify path "<A>" "<B>"` |
| Focused explanation of one concept or symbol | `graphify explain "<concept>"` |
| Blast radius of a change | `graphify affected "<symbol>"` |
| Architectural hubs | `graphify god-nodes` |
| Broad navigation | `graphify-out/wiki/index.md` |
| Broad architecture review | `graphify-out/GRAPH_REPORT.md` |

Rules:

- Start with `graphify query`. Read files only after the graph has told you which files matter.
- Run `graphify update .` after modifying code so the graph stays current (AST-only, no API cost).
- Strict mode is enabled in `.claude/settings.json`: the first raw file read of a session is
  blocked until one `graphify query` has run. Toggle it off with `GRAPHIFY_HOOK_STRICT=0`.
- If `graphify-out/` does not exist, build it once with `/graphify .` — or fall back to the text
  search tools below.

## Text search fallback

Use these when there is no graph, or when the target is not code the graph indexes — plain-text
matches in config, logs, fixtures, lockfiles, migrations by timestamp, or a literal string you
already know verbatim.

- **`rg`** instead of `grep -r` — faster, `.gitignore`-aware by default, no need to exclude
  `node_modules` or build dirs
- **`fd`** instead of `find` — shorter syntax, `.gitignore`-aware by default
  - `fd -e rb` to find by extension
  - `fd -t f` to restrict to files only

## Git diffs

- Always use **`git diff`** as-is — delta is configured as the pager and will format output automatically
- Line numbers in diff output are reliable references; use `file.rb:42` format when citing changed lines

## Security review

- When asked for a security review, run **`semgrep --config=auto .`** first and report its findings before adding your own analysis
- Semgrep findings are deterministic — treat them as facts, not suggestions
- Use focused rulesets when relevant: `p/secrets`, `p/owasp-top-ten`, `p/xss`, `p/sql-injection`

# Releases

Each release in the private repo has a git tag and a written note: goal, decisions, method, tools, measured results and next steps.

| Version | Date | Summary |
|---|---|---|
| v1.0.0 | May–Oct 2026 | Built solo in notebooks over about four weeks of spare time. Local LLM (Gemma via Ollama) for text-to-SQL and folder tags; top-level folder scan; separate database and folder results. Used by coworkers and well received |
| v2.0.0 | 2026-11-03 | Rebuild with Claude Code. No LLM; full Access import; 3-level crawl of 17 shares; folder ↔ project linking with a review list; folder briefs; hybrid FTS5 + embedding search |
| v2.0.1 | 2026-11-03 | Full review before going live: clear error replies, a private-reply fallback, tests through Slack's own routing |
| v2.0.2 | 2026-11-03 | Time-zone handling made robust on Windows Server (`tzdata`, with a fallback) |
| v2.0.3 | 2026-11-03 | Live feedback: a divider starts every bot message |
| v2.0.4 | 2026-11-03 | EC2 checklist for the files git doesn't carry; `setup.bat` reports what's missing |
| v2.0.5 | 2026-11-03 | Admin activity report: summary, bug reports, CSV export |
| v2.0.6 | 2026-11-03 | Full code and docs review: crawl, linking and override refinements; an automatic link checker for the docs |
| v2.0.7 | 2026-11-03 | Structure cleanup: dev requirements, `.env.example`, folder map |
| v2.0.8 | 2026-11-03 | Verbatim prompt log added, and keeping it current became a rule |
| v2.1.0 | 2026-11-03 | Simpler commands: a "Show all N matches" button replaces `-all`; `/purge` needs no confirmation flag |
| v2.1.1 | 2026-11-03 | Slack app home page and command hints rewritten for v2 |
| v2.1.2 | 2026-11-03 | FAQ from the office library and standards shares: 15 → 50 entries |

## Next
- **Smarter ranking:** exact-phrase ranking, a "why it matched" line under each result, and typo tolerance.
- **Operations:** a Windows service on the cloud server, a nightly rebuild, and feedback posted to an admin channel.

# The story: v1 → v2

## The problem
The office has decades of project work spread across 17 network shares: about 2.6 million files and 12.7 TB. The folders are named inconsistently: project number plus name, or a year, a proposal code, a drawing-set number, or nothing useful at all. A hand-edited Access database holds the official record of each project (about 560 projects, over 100 columns of facts), but nothing connected it to the folders.

So the everyday question, "where is project X, and what's in there?", meant asking someone who remembered, or clicking through servers.

## v1: built alone, on spare time (May–June 2026)
I built the first bot by myself, on and off over about four weeks, in Jupyter/Colab notebooks.

**Approach: two separate pipelines.**
- **Project database.** Export Access to SQLite, cut it down to a 70-column table, add an FTS5 index over 6 fields.
- **Network folders.** Scan the top level of 17 shares (1,268 folders). A local LLM (Gemma via Ollama) guessed each folder's sector and keywords from its *name*. Sentence embeddings were stored in `.pt` files.
- **Bot.** Slack Bolt in Socket Mode with `/project` (two separate result lists), `/query` (the LLM wrote SQL from a plain-English question), `/pfacts`, `/faq`, `/bug` and `/purge`.

**It worked, and people used it.** From May to October 2026, coworkers used it for project lookups, fact sheets and FAQs, and it was well received. `/project` was the most-used command, followed by `/query` and `/pfacts`.

**What it taught me.**
- **The LLM was the weakest link.** Its SQL and its folder tags were guesses, stated as confidently as facts. It was slow and needed a machine that could run a model, which ruled out a simple cloud server.
- **Keeping the two sources separate was the right start, and joining them was the next step.** Users saw database rows and folders as two lists; linking them would give one answer per project.
- **A top-level scan was enough to find folders, but not to describe them.** Going deeper would reveal projects stored inside "container" folders and say what each folder holds.
- **The data prep was a series of notebook steps.** That was fine for a prototype, but I wanted one command that rebuilds everything from scratch.

## v2: a rebuild with Claude Code (2–3 Nov 2026)
I gave an AI coding agent the v1 folder, read-only, and set the direction. Every major decision was mine; the agent analysed, proposed options, built and tested.

**Day 1: analysis before code.** I asked for background first, not code. The agent profiled the Access database and v1's scripts, then tested linking v1's folder list to projects by number and fuzzy name. It also tested whether a small ML classifier could replace the LLM's tags. Conclusion: **link folders to the real records first, and use ML only where it's confident.** The design and data documents were written before any code.

**My decisions:**
- **No LLM at all.** I dropped `/query` and the Gemma tags. The bot should only say what the data says.
- **Access is the only source of truth**, plus a fresh crawl. Nothing is reused from v1's CSV or Excel files.
- **Exhaust the data, hide nothing**: every table, every column, every saved query; every folder 3 levels deep. I had the agent explain crawl depth with a worked example before choosing.
- **Folders that refuse access stay private.** The crawl skips them and stores nothing from inside.
- **Build locally, run in the cloud.** The knowledge base is built on the office network; the bot runs on AWS.
- **Never start the live bot** from the agent's side. I run it myself.

**Day 2: build, live test, ship.** v2.0.0 landed with the crawler, the Access import, folder ↔ project linking with a review list for anything uncertain, a brief of each folder's contents, hybrid search and the bot. Then came quick improvements from live use, each a tagged release:
- Clearer messages in Slack, with a divider at the top of each one (v2.0.3).
- A one-command setup for the cloud server that checks everything it needs (v2.0.4).
- An admin activity report with a CSV export (v2.0.5).
- Simpler commands: a big green "Show all N matches" button instead of a flag to remember, and a one-step `/purge` (v2.1.0).
- An FAQ grown from 15 to 50 entries, drawn from the office's library and standards material (v2.1.2).

**Live the same day.** v2 went into use across the office on 2026-11-03. On day one, 7 people made 62 requests, and every command and button got used.

## Why v2 is better
| | v1 | v2 |
|---|---|---|
| Answers come from | Data + LLM guesses | Data only |
| Folders seen | 1,268, top level only | 1,592, incl. 321 inside containers; 3 levels deep per folder |
| Database ↔ folders | Two separate lists | Linked at build time: 699 folders → 359 projects, uncertain cases sent to a review list |
| "What's inside?" | — | A brief per folder: files, size, years, phases, sub-folders, file types, consultants |
| Searchable fields | 6 | Every registry field, sub-folder names and file-name words |
| Rebuild | Notebook steps, run by hand | `build.bat`: crawl 5 min + build 76 s, swapped in without a restart |
| Heavy dependencies | PyTorch, Ollama, pandas | None: ONNX on the CPU, no GPU |
| Tests | — | 37, including Slack's own routing against a fake Slack |
| Documentation | — | Design, data, operations and deployment docs; a note per release; a handoff file |

## What I learned
- **Not every AI project needs an LLM.** Here the useful "intelligence" was linking, structure and good search. Taking the LLM out made the bot more trustworthy, faster and cheaper to run.
- **Shipping v1 mattered.** v1's usage log told me what to keep (`/project`, `/pfacts`) and what to cut (`/query`).
- **An agent goes as far as its rules and records let it.** A rules file, a handoff document and a note per release kept the agent consistent across sessions, and kept me in charge of the decisions.

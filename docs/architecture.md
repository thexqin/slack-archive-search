# Architecture

## Goals
| Goal | What it means |
|---|---|
| No LLM | Every answer comes from the knowledge base. Nothing is generated |
| Find the folder, say what's in it | Retrieval only, like RAG without the generation step |
| One source of truth | The hand-edited Access database. The crawl adds the folders |
| Hide nothing | Every table, column and saved query; every folder and file name 3 levels deep |
| …except restricted folders | Any folder that refuses access is skipped, and nothing inside it is stored |
| Simple to run | Four `.bat` files on Windows; no Docker, no vector database, no inbound ports |

## Pipeline
```text
 Access DB ──copy──► access_import ─┐
 (hand-edited)                      │
                                    ▼
 17 shares ──► crawl ──► crawl.db ──► link ──► brief ──► index ──► knowledge base ◄── bot ◄──► Slack
               3 levels   raw         folders  per-      FTS5 +    one SQLite file,    Socket Mode
               16 threads 247 MB      roles    folder    vectors   swapped in
               ~5 min                 links    summary             atomically
```
- **Crawl and build are separate.** Editing the Access database or a link override needs only the 76-second rebuild, not a new crawl.
- **Atomic swap.** The build writes `*.db.new` and renames it into place. The bot notices the new file and reloads its vectors, with no restart and no half-built reads.
- **Three databases:** the raw crawl (rewritten each crawl), the knowledge base (rewritten each build), and an activity log (append-only, never rebuilt).

## Stages
**1. Crawl.** `os.scandir` over UNC paths, one top-level folder per task, 16 threads. It records metadata only (names, sizes, dates) and never opens a file. Junk such as `Thumbs.db`, `~$` lock files and `$RECYCLE.BIN` is skipped. Folders 4 levels down are recorded by name only, without being opened.

**2. Access import.** The `.accdb` file is copied, then every table is read into `acc_<table>` with real types. Every saved query is run, and those that return project numbers become tags on those projects.

**3. Folders and roles.** One folder per top-level folder, plus one per child of a *container* (a folder that holds many projects). A container is detected from numbered children, from children whose names match projects, or from a short hand-kept list. Each folder gets a role from its name and share: project folder, proposal, drawing set, shop-drawing scans, FTP, superseded, backup, temp, duplicate.

**4. Linking folders to projects**, tried in this order:
1. the exact folder name stored in the database;
2. the project number in the name, checking that year-like numbers aren't years;
3. alternate project numbers;
4. a strong fuzzy name match (RapidFuzz ≥ 92);
5. a hand-edited override file, which always wins.

Anything uncertain (an unknown number, names that disagree, a single-word match) goes to a review list instead of being linked automatically. A wrong link is worse than a missing one.

**5. Briefs.** For each folder: files, size and years; the largest sub-folders; the file-type mix; phases found in folder names (schematic design, design development, construction documents, competition, RFP…); notable folders (renderings, photos, specs, contracts); consultants found under team folders.

**6. Index.** One search document per project (567) and per folder (1,592), 2,159 in all:
- **Keyword index:** SQLite FTS5 with BM25 over 6 weighted fields (title, names, place, sub-folder names, file-name words, everything else).
- **Meaning index:** all-MiniLM-L6-v2 run through `fastembed` on ONNX Runtime, on the CPU. One 384-number vector per document, built from a short text card, stored in SQLite: 3.3 MB in all, about 8 s to build. At this size, a vector database isn't needed.

## Search
Each query runs three methods, and their results are fused:
1. **Exact number.** A bare project number returns only that project.
2. **Keywords.** FTS5 with AND, falling back to OR when AND finds nothing; up to 200 hits. Search operators in user input (`and`/`or`/`not`) are treated as plain words.
3. **Meaning.** One cosine-similarity pass over all vectors in memory (about 1 ms); up to 100 hits with similarity ≥ 0.45.
4. **Fusion.** Reciprocal-rank fusion, then grouping by project, so a project and its folders add up. A result found by both engines rises to the top.

Worked example, `concert hall`: the keyword engine finds 22 documents containing both words; the meaning engine finds 8 above the threshold. The top result is found by both, through its project record *and* its folder. Keywords also find proposal folders whose sub-folders mention concert halls, while meaning finds a drawing-set folder from its name alone. The 0.45 similarity threshold is tuned to keep loosely related matches out.

If the model can't load, search carries on with keywords only and logs a warning.

## The bot
Slack Bolt in Socket Mode, so it needs no public URL on the build PC or on EC2.

| Command | What it does |
|---|---|
| `/project <words or number>` | Top 5 results, plus a green **Show all N matches** button. Each result has **Inside** (the folder brief) and **Project facts** buttons |
| `/pfacts <project>` | Every non-empty database field, uploaded as a file |
| `/faq <question>` | Semantic match over 50 office FAQ entries |
| `/bug <text>` | Logged for the admin report |
| `/purge` | In a DM, deletes all of the bot's messages and files, never the user's own |

Every command and button is written to the activity log; `activity.bat` turns it into a summary, a list of bug reports, or a CSV.

## Choices I rejected
| Option | Why not |
|---|---|
| LLM for answers or tags | Guesses presented as facts; slow; needs a GPU machine |
| Reusing v1's CSV and Excel files | Stale, and partly LLM guesses; a fresh crawl takes 5 minutes |
| Incremental crawl | Not worth the complexity when a full crawl takes 5 minutes |
| Vector database | About 2,000 vectors fit in memory |
| Docker | Windows Server, with Access read through ODBC |
| PyTorch | ONNX Runtime runs the same model on the CPU with a far smaller install |

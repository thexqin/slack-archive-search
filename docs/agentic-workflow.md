# How I worked with an AI coding agent

v2 was built with **Claude Code** over two days (2–3 Nov 2026): 36 requests, 12 tagged releases, live for the office on day two. I directed the work and made the decisions; the agent analysed, wrote the code, tests and docs, and reported back with measurements.

## Who did what
| Me | The agent |
|---|---|
| Set the goal and the constraints | Profiled the data and v1's code; wrote the analysis docs |
| Chose between options (crawl depth, no LLM, sources of truth) | Laid out the options, with worked examples and measured trade-offs |
| Ran the live bot and reported what I saw | Fixed what I reported, added a test, cut a release |
| Asked the hard questions ("what is your scoring system?") | Measured the answer and documented it |
| Owned the Slack app, the secrets and the EC2 server | Never touched the live bot or the real token |

## The guardrails
**1. A rules file the agent reads first** (`CLAUDE.md`, abridged):
```text
Start by reading docs/handoff.md and the latest versions/vX.Y.Z.md.

Hard rules:
- Never modify anything under previous/ (read-only v1 reference, not in git).
- No LLM features. The Access database is the only data source besides the fresh crawl.
- Restricted (access-denied) folders: never record or show them.
- Every change of substance: write versions/vX.Y.Z.md, bump the version,
  update the handoff, commit, tag, push.
- Every owner request: append it to docs/chat-history.md, verbatim,
  with a 1–3 line summary of the response, in the same commit as the work.
```
A separate memory note added: *never start the bot; tests use a dummy token and a local fake Slack.*

**2. A handoff document**, so any session can continue with no chat history: what the system is, my standing rules, current state, what's done and not done, open questions for me, next steps in order, and a dated timeline of the work.

**3. A note per release**, from a template: goal, decisions and **who made each one**, method, tools and versions, measured results, next steps. Git records *what* changed; these notes record *why*.

**4. A verbatim prompt log.** Every request I made, typos included, with a short summary of what was done. It's the honest record of how the system came about.

**5. Tests before the live run.** Before I first ran the bot in Slack, I asked for a full review. The agent hardened every command and button and added tests that run Slack's own routing against a fake Slack. Every later change came with its own test.

## Excerpts from the prompt log
Verbatim, typos included. `[…]` marks cuts and replaced names.

**The opening brief (#1)**
```text
modernize this project. it's a SLACK app bot for our office coworkers around 50 people. i plan to drop the ollama feature as the whole bot will be running on aws ec2. reference the "previous" folder but do not change the file inside. lay out the design you would like to make. also make sub folder for all documentations in md.
```

**Stepping back to understand the data first (#2)**
```text
let's step back. we need some background info on the project. there two source of truth. […] I did run a second path on all project folder names with ollama and local gemma4 LLM to infer the project category, sector, sub sector, keywords. those are inferred from project folder names only. can be wrong. […] is it possible to merge the two sources? […] ideally we should start without LLM, and use machine learning or python, sqlite tools and semantic search. first document everything. then suggest the best approach.
```
*Agent:* profiled the database, test-linked v1's folder list to projects, and tested a no-LLM classifier. Wrote four data documents before any code.

**Checking we mean the same thing (#4)**
```text
clarify the crawl depth by using an example so we know if we are on the same page
```

**The key decision (#5)**
```text
assume option A. also get rid of query. we will not use LLM. this is more like a rag experiment in a way that we index each project folder with everything we can find 3 levels deep into somesmall but useful information. we let users query and find the project folder they want and give them a brief of what's inside the folders. […]
```
*Agent:* built v2.0.0: crawl of 17 shares (2.57 M files in 5 min), knowledge base, hybrid search, bot, tests, docs.

**Keeping control of production (#9)**
```text
review the code once again and we are ready to talk to slack for a test. never start the bot on your end. i'll manually do run.bat
```

**Polishing from live use (#12)**
```text
make sure there is dividers to each message. right now messages on screen look glued to each other and hard to tell message 1 from 2 from 3.
```

**Challenging the agent's design (#25)**
```text
so you never merged the two sources. just show them at the result step? what is your scoring system and how do you limit the results
```
*Agent:* showed that the sources are merged at build time (folders inherit project data, and projects carry their folder names), then documented the scoring, grouping and limits.

**Designing for real users (#28, #30)**
```text
people always find the syntax a little difficult to remember. like "/project -all search" it's easy to type the wrong command and does space matter there?
```
```text
let's do a new version then. only project pfact purge(no yes and delete ALL bot messages), faq and bug. this way it's easy to remember. for project, return the default search, and another very visible button to let user get the "project -all" experience
```
*Agent:* v2.1.0, with a big green "Show all N matches" button. The old `-all` spellings still work and are never searched as words.

## What I'd tell someone starting out
- **Ask for analysis and documents before code.** The first day produced no code, and it's why v2 holds together.
- **Write your rules down once.** A rules file beats repeating yourself in every session.
- **Make the agent measure.** "Does X work?" got numbers back, not opinions.
- **Keep production in human hands.** The agent never ran the live bot or saw the real token.
- **Keep a record a stranger could follow.** The handoff, the release notes and the prompt log mean the work survives the chat.

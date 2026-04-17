---
name: x-wiki
description: Build and maintain a project knowledge base. Use when the user asks to "build the wiki", "update the wiki", "brief me", "what do we know about X", "how does X work in this project", "update wiki from done-log", "check wiki for stale info", "/x-wiki build|update|query|lint|help", or any request to document, retrieve, or verify accumulated project knowledge. Also triggers on re-onboarding phrases like "I'm coming back to this project", "refresh my memory", "give me an overview". This skill reads from repo code, git history, done-logs, and web searches to compile and maintain structured wiki pages. Always use this skill for project knowledge questions before analyzing code from scratch — the wiki may already have the answer.
---

# x-wiki — Project Knowledge Base

A skill for building and maintaining a persistent, compounding project knowledge base.
Inspired by Karpathy's LLM Wiki pattern, adapted for software development workflows.

## Core Concept

Instead of re-analyzing code from scratch every session, the LLM compiles knowledge
into persistent wiki pages that accumulate over time. The wiki sits between you and
the raw sources (code, git history, done-logs). Knowledge is compiled once, kept
current, and queryable across sessions.

**You own:** choosing what to document, asking questions, steering direction, verifying accuracy.
**The LLM owns:** all writing, cross-referencing, source-checking, and maintenance.

## Base Path

This skill is part of x-suite. The base path is the x-suite working directory
defined in CLAUDE.md (default: `.work/`). This skill's store is at `<base>/wiki/`.
Done-logs from x-done-log are at `<base>/done/`.

Throughout this document, `<base>` refers to this configured path.

## Architecture

Three layers (read-only sources → compiled wiki → this skill as schema):

```
Source Layer (read-only, never modified by this skill):
  - Repo code and config files (working tree)
  - Git history (commits, diffs, blame)
  - <base>/done/*.md (task records from x-done-log)
  - Web searches (performed during sessions)

Wiki Layer (LLM-written, lives at <base>/wiki/):
  <base>/wiki/
  ├── index.md             ← page catalog: every page with one-line summary
  ├── log.md               ← append-only operation log
  ├── overview.md           ← project architecture, structure, the map
  ├── glossary.md           ← terms, servers, DBs, URLs, service accounts
  ├── recipes.md            ← how-to procedures (deploy, debug, dev setup)
  ├── tribal.md             ← gotchas, "why" decisions, non-obvious traps
  └── areas/                ← deep pages per topic, created on demand
      ├── database.md
      ├── deployment.md
      └── (created as needed)

Schema Layer:
  - This SKILL.md file (governs conventions and workflows)
```

## Operations

### build — Create initial wiki from repo analysis

Trigger: `/x-wiki build`, "give me an overview of this project", "build the wiki",
or when `<base>/wiki/` does not exist and user asks any project knowledge question.

Steps:
1. Create `<base>/wiki/` directory structure
2. Analyze repo: directory structure, entry points, config files, dependencies,
   build/deploy scripts, README, CLAUDE.md
3. Analyze git history: recent activity, major contributors, active areas
4. Read any existing `<base>/done/` logs for prior knowledge
5. Write `overview.md` — project architecture and structure
6. Write `glossary.md` — extract terms, server names, DB names, URLs, accounts
7. Write `recipes.md` — extract any deployment/setup/debug procedures found
8. Write `tribal.md` — extract any gotchas or decisions from done-logs
9. Create area pages under `areas/` for major subsystems identified
10. Write `index.md` — catalog of all pages with one-line summaries
11. Write initial `log.md` entry

### update — Integrate new knowledge into wiki

Trigger: `/x-wiki update`, "update the wiki", "add this to the wiki",
"capture what we learned", "update wiki from done-log".

Input sources (in priority order):
1. Current session context (if task was just completed)
2. Specific done-log file (if user names one)
3. Most recent done-log (if user says "last done-log" or "latest")
4. Multiple done-logs (if user says "this week", "recent", etc.)

Steps:
1. Identify the input source(s)
2. Read the source material
3. Read `index.md` to find which existing wiki pages might be affected
4. For each affected page:
   a. Read the page
   b. Check the `sources:` frontmatter — verify referenced files still match current code
   c. Update content with new knowledge
   d. Update `sources:` frontmatter if new files are relevant
   e. Update `last-verified:` date
5. If knowledge doesn't fit an existing page, create a new area page
6. Update `index.md` with any new or changed pages
7. Append to `log.md`
8. Report what was updated and what files were touched

Cross-reference check: After updating, look at the files touched in the done-log.
If other wiki pages reference those same files (check `sources:` frontmatter),
flag them: "areas/database.md also references server.xml — want me to review it?"

### query — Answer questions from wiki, verify against code

Trigger: `/x-wiki query <topic>`, "how does X work", "what do we know about Y",
"brief me on Z", "refresh my memory on X".

Steps:
1. Read `index.md` to find relevant pages
2. Read the relevant wiki page(s)
3. **Always verify**: check `last-verified` date. If older than 2 weeks or if the
   topic is config/code-dependent, cross-check key claims against current source files
   listed in the page's `sources:` frontmatter
4. If current code diverges from wiki:
   a. Update the wiki page with corrected information
   b. Mark what changed and why
   c. Update `last-verified:` date
   d. Append to `log.md`
   e. Tell the user what changed: "Note: wiki said X but code now shows Y, updated."
5. Answer the user's question based on (now-verified) wiki content
6. Cite sources: wiki page path + original source references

If no relevant page exists:
1. Analyze the code/config directly to answer the question
2. Write a new wiki page with the findings
3. Update `index.md`, append to `log.md`
4. Answer the user

### lint — Health check the wiki

Trigger: `/x-wiki lint`, "check the wiki", "is the wiki still accurate",
"wiki health check".

Checks to perform:
1. **Staleness**: scan `last-verified` dates, flag pages older than 30 days
2. **Source drift**: for each page, check files in `sources:` frontmatter —
   have they been modified since `last-verified`? Use `git log --since` on those files
3. **Orphan detection**: pages not linked from any other page
4. **Coverage gaps**: scan recent `<base>/done/` logs for topics that have no wiki page
5. **Missing cross-references**: concepts mentioned in pages that should link to
   other existing pages but don't
6. **Contradictions**: claims in one page that conflict with another
7. **Glossary completeness**: terms/servers/URLs used in code but not in glossary.md

Output: a lint report listing findings by severity (stale, drifted, missing, inconsistent).
Offer to fix each finding. Append lint run to `log.md`.

### help — Show usage

Trigger: `/x-wiki help`, `/x-wiki`

Output the following:

```
x-wiki — project knowledge base

commands:
  /x-wiki build                     analyze repo, create initial wiki
  /x-wiki update                    update wiki from current session
  /x-wiki update from last done-log update from most recent done-log
  /x-wiki update from done-logs this week
  /x-wiki query <topic>             look up what we know about <topic>
  /x-wiki lint                      check for stale/missing/inconsistent pages
  /x-wiki help                      show this help

natural triggers:
  "how does X work in this project"  → query (checks wiki first)
  "give me an overview"              → build (if no wiki) or query overview
  "update wiki with what we learned" → update from session
  "I'm coming back to this project"  → query overview for re-onboarding

wiki location: <base>/wiki/ (base path from CLAUDE.md, default .work/)
sources: repo code, git history, <base>/done/, web, session context
```

## Wiki Page Format

Every wiki page follows this format:

```markdown
# Page Title
<!-- last-verified: YYYY-MM-DD -->
<!-- sources: path/to/file1.ext, path/to/file2.ext -->

Content here. Each factual claim should have an inline source reference.

The application connects to DB2 via JNDI lookup in server.xml.
[src/main/config/server.xml:42-58]

Connection pool was increased from 20 to 50 after timeout incidents.
[a1b2c3d server.xml, done/2026-03-12-002.md]

WebSphere Liberty is configured with features javaee-8.0 and jdbc-4.2.
[src/main/config/server.xml:3-8]
```

### Source reference conventions

- **From code/config (current state):** `[path/to/file.ext:line-range]`
  Use repo-relative paths. Line range is optional but preferred for precision.
- **From git history (changes/decisions):** `[commitsha path/to/file.ext]`
  Use short hash (7 chars). Use for "why" knowledge — when/why something changed.
- **From done-logs:** `[done/YYYY-MM-DD-NNN.md]`
  Reference the done-log file by name.
- **From web:** `[url, accessed YYYY-MM-DD]`
- **Combined:** `[a1b2c3d server.xml, done/2026-03-12-002.md]`
  When a claim is supported by multiple sources.

### index.md format

```markdown
# Wiki Index
<!-- last-updated: YYYY-MM-DD -->

## Core
- [overview.md](overview.md) — project architecture, structure, tech stack
- [glossary.md](glossary.md) — terms, servers, databases, URLs, accounts
- [recipes.md](recipes.md) — how-to procedures: deploy, debug, dev setup
- [tribal.md](tribal.md) — gotchas, "why" decisions, non-obvious traps

## Areas
- [areas/database.md](areas/database.md) — DB2 and Netezza configuration, connection pooling
- [areas/deployment.md](areas/deployment.md) — CI/CD pipeline, WebSphere Liberty deployment
```

### log.md format

Append-only. Each entry is parseable with grep.

```markdown
# Wiki Log

## [2026-04-16] build | Initial wiki creation
Analyzed repo structure, created overview.md, glossary.md, recipes.md, tribal.md.
Created area pages: areas/database.md, areas/deployment.md.
Source: repo analysis + 12 done-logs.

## [2026-04-16] update | Connection pool fix
Updated areas/database.md with pool size change (20→50).
Updated tribal.md with timeout gotcha.
Source: done/2026-04-16-001.md

## [2026-04-17] query | auth flow (verified, no changes)
Answered query from areas/auth.md. Cross-checked against current code, no drift.

## [2026-04-20] lint | Weekly health check
Checked 8 pages. Found: 1 stale (areas/deployment.md, 45 days),
2 source-drifted files. Fixed deployment.md. User declined other fixes.
```

## Important Rules

1. **Never modify source material.** Repo code, git history, and done-logs are read-only
   inputs. The wiki is the only thing this skill writes.
2. **Always cite sources.** Every factual claim in a wiki page must have an inline
   source reference in brackets.
3. **Always verify on query.** When answering from the wiki, cross-check against
   current code if the page's sources are code files. Update the page if drifted.
4. **Accumulate, don't rewrite** for tribal.md and log.md — append new entries.
   But overview.md, glossary.md, and area pages should be rewritten to stay current.
5. **Create area pages on demand.** Don't pre-create empty pages. When knowledge
   first emerges for a topic that deserves its own page, create it then.
6. **Update index.md on every write operation.** The index is how future sessions
   find relevant pages without reading everything.
7. **Append to log.md on every operation.** Build, update, query (with verification
   result), and lint all get logged.

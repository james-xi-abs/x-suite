# x-suite

A personal skill library for Claude Code CLI, built for software development workflows.

## What Is This

x-suite is a collection of Claude Code skills that help with project knowledge management,
task documentation, and development workflows. Each skill is a standalone folder with a
SKILL.md that Claude Code reads and follows when triggered.

The design is inspired by Karpathy's LLM Wiki pattern, adapted for hands-on software
development: instead of re-analyzing code from scratch every session, knowledge is compiled
into persistent markdown files that accumulate over time. You stay in control — these are
manually invoked tools, not an autonomous agent loop.

## Principles

- **Skills, not agents.** You drive, Claude executes. No auto-loops, no magic.
- **Plain markdown, plain files.** Everything is readable, greppable, git-diffable.
- **Compile once, query forever.** Knowledge is built up, not re-derived every session.
- **Source-linked.** Every claim traces back to code, commits, done-logs, or URLs.
- **Portable.** Copy a skill folder to any machine with Claude Code and it works.

## Skills

| Skill | Description |
|---|---|
| [x-wiki](./x-wiki/) | Project knowledge base — build, query, update, and lint a persistent wiki compiled from repo code, git history, and done-logs |
| [x-done-log](./x-done-log/) | Task record — write a structured retrospective after completing a bug fix, feature, or investigation |

## Setup

### 1. Install skills

Copy or symlink the skill folders to your Claude Code skills path:

```bash
# Clone the repo
git clone <your-x-suite-repo-url>

# Option A: symlink individual skills
ln -s /path/to/x-suite/skills/x-wiki ~/.claude/skills/x-wiki
ln -s /path/to/x-suite/skills/x-done-log ~/.claude/skills/x-done-log

# Option B: symlink the whole skills folder
ln -s /path/to/x-suite/skills ~/.claude/skills/x-suite
```

### 2. Configure base path

All x-suite skills share a common working directory (default: `.work/`). Each skill
stores its data in a subfolder (e.g. `.work/done/`, `.work/wiki/`).

To configure, open Claude Code in your project and run:

```
add this to .claude/CLAUDE.md:

# x-suite
x-suite working directory: .work/
```

Replace `.work/` with any path you prefer (e.g. `.x-suite/`, `.local/`, etc.).

This goes in `.claude/CLAUDE.md` (personal, gitignored) so the wiki data stays private.
If you want to share the config with your team, put it in `./CLAUDE.md` (project root, in git) instead.

### 3. Gitignore the working directory

```
add .work/ to .gitignore
```

## Usage

### x-wiki

```
/x-wiki build                      analyze repo, create initial wiki
/x-wiki update                     update wiki from current session
/x-wiki update from last done-log  update from most recent done-log
/x-wiki query <topic>              look up what we know about <topic>
/x-wiki lint                       check for stale/missing/inconsistent pages
/x-wiki help                       show help
```

Or use natural language — Claude triggers the skill automatically:

```
"give me an overview of this project"
"how does the database config work?"
"I'm coming back to this project, brief me"
"update wiki with what we learned"
```

### x-done-log

```
/x-done-log
```

Or natural triggers:

```
"write a reference file for future sessions"
"summarize what we did"
"log this fix"
```

### Typical Workflow

```
1. Start on a project         →  /x-wiki build (first time)
                                  /x-wiki query overview (returning)

2. Work on a task              →  (normal development)

3. Finish the task             →  /x-done-log

4. Capture knowledge           →  /x-wiki update
                                  or: /x-wiki update from last done-log

5. Later, need to look up      →  /x-wiki query <topic>
   how something works

6. Periodic maintenance        →  /x-wiki lint
```

## Repo Structure

```
x-suite/
├── README.md              ← this file
├── skills/                ← Claude Code skills
│   ├── x-wiki/
│   │   └── SKILL.md       ← project knowledge base skill
│   ├── x-done-log/
│   │   └── SKILL.md       ← task record skill
│   └── (future skills)/
│       └── SKILL.md
└── (future: mcp/, tools/, templates/ as needed)
```

Each skill is self-contained in its folder under `skills/`. The only shared convention
is the base path configured in CLAUDE.md.

## Adding New Skills

Any skill prefixed with `x-` is part of x-suite. To add a new skill:

1. Create a folder: `x-<name>/SKILL.md`
2. In the SKILL.md, include: "This skill's store is at `<base>/<name>/` where `<base>` is the x-suite working directory defined in CLAUDE.md (default: `.work/`)."
3. That's it.

## Background

This project draws from several ideas:

- **Karpathy's LLM Wiki** — compile knowledge into persistent markdown instead of re-deriving it via RAG on every query. Raw sources are read-only, the wiki is LLM-maintained, a schema governs conventions.
- **OpenSpec** — everything is inspectable. Wiki pages cite their sources, every operation is logged, page freshness is tracked.
- **GSD (Get Stuff Done)** — bias toward action. Done-logs capture what was actually done, not what was planned. The wiki compiles real experience, not theory.
- **Hermes Self-Evolution** — the wiki improves over time. Lint catches staleness, updates compound knowledge, the system gets more valuable with use.
- **Mempalace** — structured persistent memory across sessions. The wiki is organized by type (overview, tribal, recipes, glossary, areas) for different retrieval patterns.

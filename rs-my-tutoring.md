# Archon Tutoring Session — 2026-03-21

## Original Plan: Session Topics (3 layers)

### 1. The App Itself (~10 min)

- Web UI is new — localhost:5173 gives you a chat interface
- SQLite is default now (zero setup)
- Workflow DAG execution is new (parallel node execution with conditions)

### 2. The .claude/ Toolbox (~15 min)

- Agents (24 specialized prompt templates) — code reviewer, silent-failure-hunter, type-design-analyzer, etc.
- Skills (14 slash commands) — /archon, /release, /commit, /validate, etc.
- Commands — prompt templates for Claude Code tasks
- These are Claude Code features, not Archon features — they work in any repo

### 3. Using Tools in Other Projects (~10 min)

- /copy-tools /path/to/target — copies agents, commands, workflows to another repo
- .archon/workflows/ and .archon/commands/ in any repo — Archon discovers them automatically
- CLI: bun run cli workflow run <name> "prompt" from any git repo

---

## Learnings So Far

### Local Setup

- Minimal .env: just `CLAUDE_USE_GLOBAL_AUTH=true` and `DEFAULT_AI_ASSISTANT=claude`
- SQLite auto-creates at `~/.archon/archon.db` — delete old DB if schema is outdated
- `bun run dev` starts backend (port 3090) + Web UI (port 5173)
- No platform tokens needed — Web UI works standalone

### Architecture Shift (old 3-package → new 9-package monorepo)

- Old: core, cli, server (adapters and isolation bundled inside)
- New: paths, git, isolation, workflows, core, adapters, server, web, cli
- Each package has a focused responsibility with clear dependency direction
- Web UI (React + Vite + shadcn/ui) is entirely new

### Project Registration (/clone)

- More than a DB row — creates full project structure:
  `~/.archon/workspaces/<owner>/<repo>/{source, worktrees, artifacts, logs}`
- Local paths get symlinked (not copied)
- Auto-detects assistant type (.claude/ vs .codex/ folder)
- Auto-loads commands from .archon/commands/
- Registration is permanent; worktrees are temporal

### Workspace vs Worktree

- **Workspace**: permanent project home in ~/.archon/workspaces/
- **Worktree**: temporary isolated git branch checkout for a single task
- Multiple worktrees can run in parallel on the same project
- Cleanup: by age (`isolation cleanup 7`), by merge status (`--merged`), or explicit (`complete <branch>`)

### Three-Layer Config System

```
Layer 1: Bundled defaults (compiled into app)
  └── .archon/workflows/defaults/  — 16 workflows
  └── .archon/commands/defaults/   — 50+ commands
  Always available, no copying needed.

Layer 2: Global (~/.archon/.archon/)
  └── workflows/  — personal workflows
  └── commands/   — personal commands
  Overrides Layer 1 by filename.

Layer 3: Project repo (<repo>/.archon/)
  └── workflows/  — repo-specific workflows
  └── commands/   — repo-specific commands
  Overrides Layers 1 and 2 by filename.
```

### Claude Code vs Archon — Different Systems

- **Claude Code** (this CLI): Anthropic's coding assistant, reads files directly, interactive
- **Archon** (Web UI / Archon CLI): workflow orchestrator, manages AI via SDK, creates worktrees
- `.claude/` directory = Claude Code features (agents, skills, commands) — repo-local only
- `.archon/` directory = Archon features (workflows, commands, config) — has global layer
- They are NOT connected — different tools for different purposes

### Git Strategy: Option B Selected

- Previous approach (Option A): rs-\* prefixed files in fork, separated by naming convention
- New approach (Option B): personal tools in global ~/.archon/ layer, repo stays clean
- `git pull upstream main` without merge conflicts
- rs-\* prefix convention retired for new work
- Mental shift: "old code and tools depreciate faster in AI coding era — lifecycle has dropped dramatically"

### Personal Tools — Best Practice (replacing rs-\* prefix convention)

The old approach was prefixing custom files with `rs-*` to separate from upstream in the same directory. The new approach: **don't fork to customize, layer instead.**

**Claude Code** (personal tools):

```
~/.claude/CLAUDE.md          ← global instructions (already exists)
~/.claude/commands/          ← personal slash commands (available in ALL repos)
~/.claude/agents/            ← personal agent definitions (available in ALL repos)
~/.claude/settings.json      ← global permissions, hooks
```

**Archon** (personal workflows):

```
~/.archon/.archon/workflows/ ← personal workflows (overrides bundled defaults)
~/.archon/.archon/commands/  ← personal commands (overrides bundled defaults)
~/.archon/config.yaml        ← global config
```

Note: `~/.archon/.archon/` looks like a stutter — it's a side effect of the discovery logic always searching for `.archon/workflows` relative to a root. The root for global is `~/.archon/`. Ugly but consistent. The community repo is still in rapid development; this naming may improve later.

**Project-level** (only when truly repo-specific):

```
<repo>/.archon/workflows/    ← repo-specific workflows
<repo>/.archon/commands/     ← repo-specific commands
<repo>/.claude/commands/     ← repo-specific Claude Code commands
<repo>/.claude/agents/       ← repo-specific agents
```

No prefixes needed. The filesystem hierarchy IS the namespace.

### Web UI Observations

- Projects list empty until you /clone or register a repo
- Workflows tab empty without a registered project (needs cwd to discover)
- Dark mode only (no light theme)
- `/codebase-switch` to change which project a conversation targets

---

### Workflow Test: archon-remotion-generate

Tested the Remotion video generation workflow with RALA/Tietoturva branding.

**What happened:**

- CLI: `bun run cli workflow run archon-remotion-generate --cwd /tmp/rala-clips --no-worktree "..."`
- DAG steps: check-project (pass) → generate (pass) → render-preview (FAIL) → skipped rest
- render-preview bash script has a parsing bug (`npx remotion compositions` outputs "6%" which breaks arithmetic)
- AI-generated code was good — 6 compositions created correctly
- Manual rendering worked fine

**Key insight: When does Archon add value?**

- NOT for one-off creative tasks with iteration — Claude Code directly is faster
- YES for repeatable processes, autonomous execution, multi-agent parallel work
- YES when you have a proven process to run unattended across multiple projects

**Practical issues found:**

- `/clone` via Web UI chat is slow (goes through AI router, not deterministic command)
- Use REST API (`curl POST /api/codebases`) or the Add Project form for registration
- Logo background removal is a manual/design step
- Sound effects need real audio assets
- Remotion Studio: compositions sidebar is collapsed by default
- `React.lazy` imports can cause blank screens in Remotion Studio — use direct imports

**Result saved to:** `/Sites/Virtanen/rsult-app01/video-production/intro-outro-generator/`

---

## Completed

- [x] Local setup (.env, bun install, type-check, dev server)
- [x] Register rca-archon in Archon (via curl API)
- [x] Reset origin/main to match upstream (Option B clean slate)
- [x] Test a workflow run (archon-remotion-generate via CLI)

## Next Steps

- [ ] Set up global directories: `~/.claude/commands/`, `~/.claude/agents/`, `~/.archon/.archon/workflows/`, `~/.archon/.archon/commands/`
- [ ] Explore .claude/ toolbox (agents, skills) — Topic 2
- [ ] Try /copy-tools to another project — Topic 3
- [ ] Finalize intro-outro-generator clips (better sounds, render MP4s)
- [ ] Parameterize generator for multi-client reuse

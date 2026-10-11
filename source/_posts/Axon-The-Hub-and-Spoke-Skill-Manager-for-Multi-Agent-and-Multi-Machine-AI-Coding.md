---
title: >-
  Axon: The Hub-and-Spoke Skill Manager for Multi-Agent and Multi-Machine AI
  Coding
date: 2026-10-11 20:55:00
tags: [AI, Agent, Skill, CLI, Developer-Tools, Architecture, Claude, Devin, Cursor, Open-Source]
---

As developers incorporate AI into their daily workflows, their toolchains rarely converge on a single client. An engineer might rely on **Claude Code** for fast terminal-driven tasks, **Devin** or **Cursor** for multi-file code editing, **Antigravity** for deep architectural analysis, and specialized CLI agents for database administration or infrastructure tasks.

Furthermore, modern engineering workflows are distributed across heterogeneous machines: a MacBook for travel, a high-performance workstation for local builds, and remote Linux servers or WSL clusters for production-like execution.

While utilizing multiple AI coding assistants maximizes flexibility, it introduces a severe operational friction: **configuration fragmentation and skill drift**.

Every AI coding environment defines its own proprietary configuration directory:
- Claude Code reads from `~/.claude/skills/`, `~/.claude/commands/`, `~/.claude/rules/`, and `~/.claude/CLAUDE.md`
- Devin reads from `~/.codeium/windsurf/skills/`, `global_workflows/`, and `memories/global_rules.md`
- Antigravity reads from `~/.gemini/config/skills/`, `global_workflows/`, and global rule configurations
- OpenAI Codex reads from `~/.codex/skills/` and `~/.codex/AGENTS.md`
- Roo Code, Cursor, Trae, Qoder, and Kiro maintain similar bespoke directories

When you engineer an effective debugging skill, slash command, or prompt rule in one editor, those improvements remain locked in that tool's directory. Over time, prompt definitions diverge across machines, community skills fall out of date, and unversioned secrets risk leaking into local files.

To solve this fragmentation, we built **[Axon](https://github.com/kamusis/axon-cli)**—an open-source, Git-backed, hub-and-spoke skill manager written in Go.

---

## 1. Architectural Foundation: The Hub-and-Spoke Model

Instead of treating each AI editor as an isolated island, Axon decouples skill storage from editor consumption through a **Hub-and-Spoke** architecture.

<p class="text-center" style="margin: 2rem 0;">
  <img src="/img/axon-hub-and-spoke.svg" alt="Axon Hub-and-Spoke Architecture" style="width: 100%; max-width: 1000px; height: auto; border-radius: 12px; box-shadow: 0 4px 20px -2px rgba(0, 0, 0, 0.08);" />
</p>

### Core Components

1. **The Hub (`~/.axon/repo/`)**: A canonical, local Git repository that stores all skills, reusable workflows, slash commands, rules, and global instructions. It serves as the single source of truth across all tools.
2. **The Spokes (Filesystem Symlinks)**: Rather than duplicating files or running background file-sync daemons, Axon establishes filesystem symlinks from each tool's expected configuration directory directly into subdirectories of the Hub.
3. **Hub Manifest (`axon.vendors.yaml`)**: An in-tree specification inside the Hub repository declaring external third-party skill repositories, automatically shared across machines via Git.
4. **The Remote Repository**: A private or public Git repository configured as the Hub's upstream origin, enabling cross-machine synchronization.

### Directory Targets vs. File Targets

Axon handles two distinct integration patterns natively:

1. **Directory Targets**: Maps entire categories (`skills`, `workflows`, `commands`, `rules`) directly to editor directories (e.g., `~/.claude/skills` $\rightarrow$ `Hub/skills`).
2. **File Targets**: Fans out a single central rulebook (`global_rules.md`) to multiple tool-specific filenames:
   - `~/.claude/CLAUDE.md` $\rightarrow$ `Hub/global_rules.md`
   - `~/.codex/AGENTS.md` $\rightarrow$ `Hub/global_rules.md`
   - `~/.codeium/windsurf/memories/global_rules.md` $\rightarrow$ `Hub/global_rules.md`

### 100% Agent-Agnostic: Starter Templates vs. Arbitrary Custom Agents

A common misconception is that Axon only works with a fixed list of mainstream tools. In reality, **Axon is completely agent-agnostic**.

The out-of-the-box configurations generated during `axon init` (such as Claude Code, Devin, Cursor, Antigravity, Roo Code) are simply **convenient starter templates**. Axon can synchronize skills, rules, and workflows for **any AI agent, CLI utility, or proprietary internal tool** you use.

Adding support for any custom agent requires only a quick entry in `~/.axon/axon.yaml`:

```yaml
targets:
  - name: custom-db-agent
    source: skills                      # Hub category: skills, workflows, commands, or rules
    destination: ~/.db-agent/skills     # Absolute path where your agent expects skills
    type: directory                     # directory or file

  - name: custom-agent-rules
    source: global_rules.md             # Canonical single source of truth
    destination: ~/.db-agent/RULES.md   # Agent-specific prompt / system instruction file
    type: file
```

Running `axon link custom-db-agent` immediately bridges the new tool to your central Hub. Whether you use commercially available tools or proprietary in-house AI agents, Axon treats them all uniformly.

### Zero-Latency Consistency without Daemon Overhead

Background file-syncing services often introduce race conditions, file-locking contention, and high CPU usage. By relying on native filesystem symlinks:
- **Zero Latency**: When an AI agent modifies a skill in any tool, the canonical file in `~/.axon/repo/skills/` updates immediately.
- **Immediate Cross-Tool Visibility**: All connected agents and editors—whether standard tools or proprietary in-house bots—read the updated content on their next access.
- **Zero Background Resource Cost**: No daemon threads, polling loops, or memory overhead.

---

## 2. External Skill Lifecycle & In-Tree Provenance

In multi-agent environments, engineers frequently import skills from external GitHub repositories—such as Anthropic's public skills, curated OpenAI prompts, or specialized community utilities.

Managing external skills using traditional Git submodules introduces fragile detached HEAD states and recursive clone hurdles. Conversely, manually copying files loses all upstream lineage. 

Axon solves this with a **complete vendor management lifecycle** coupled with **in-tree provenance tracking**.

<p class="text-center" style="margin: 2rem 0;">
  <img src="/img/axon-vendor-lifecycle.svg" alt="Axon Vendor Skill Lifecycle and Provenance" style="width: 100%; max-width: 960px; height: auto; border-radius: 12px; box-shadow: 0 4px 20px -2px rgba(0, 0, 0, 0.08);" />
</p>

### The Hub Vendor Manifest (`axon.vendors.yaml`)

Vendors are declared in `axon.vendors.yaml` located at the root of the Hub Git repository:

```yaml
# ~/.axon/repo/axon.vendors.yaml
version: 1
vendors:
  - name: book-to-skill
    repo: https://github.com/virgiliojr94/book-to-skill.git
    subdir: .
    dest: skills/book-to-skill
    ref: master
  - name: anthropic-customer-support
    repo: https://github.com/anthropics/anthropic-quickstarts.git
    subdir: computer-use-demo
    dest: skills/customer-support
    ref: v0.2.0
```

Because `axon.vendors.yaml` is version-controlled inside the Hub repo, running `axon sync` across your laptop and workstation ensures all machines stay aligned on the exact same vendor declarations.

### Declarative Lifecycle Operations

Axon provides a dedicated suite of subcommands to manage the complete lifecycle of third-party assets:

```bash
# 1. Add an external vendor (with optional subdir, ref, and destination)
axon vendor add https://github.com/anthropics/skills.git skills/skill-creator \
  --subdir skills/skill-creator \
  --ref main

# 2. Inspect configured vendors and their local sync health (offline)
axon vendor list

# 3. Check for upstream updates across all vendors without altering local disk
axon vendor check

# 4. Mirror files via sparse-checkout into the Hub
axon vendor sync

# 5. Eject a vendored skill to manage it as first-party code
axon vendor eject skills/skill-creator
```

```text
$ axon vendor list
NAME                        DEST                     REF     STATUS              REPO
book-to-skill               skills/book-to-skill     master  synced (7d2a8f10)   github.com/virgiliojr94/book-to-skill
anthropic-customer-support  skills/customer-support  v0.2.0  synced (4c1b9e22)   github.com/anthropics/anthropic-quickstarts
diagram-design              skills/diagram-design    main    missing dest        github.com/cathrynlavery/diagram-design
```

### In-Tree Provenance (`.axon-vendor.yaml`)

When Axon mirrors an external vendor, it writes a lightweight metadata file directly into the destination directory:

```yaml
# ~/.axon/repo/skills/book-to-skill/.axon-vendor.yaml
repo: https://github.com/virgiliojr94/book-to-skill.git
ref: master
commit: 7d2a8f103b4c1a2d3e4f5a6b7c8d9e0f1a2b3c4d
subdir: .
synced_at: 2026-10-11T12:00:00Z
```

This delivers two major advantages:
1. **Self-Describing Lineage**: Commands like `axon list` automatically render `[vendor]` badges, and `axon inspect <skill>` displays upstream origin, commit hash, and sync timestamp.
2. **Deterministic Skipping**: When running `axon vendor sync`, Axon compares the upstream commit SHA against the in-tree `.axon-vendor.yaml`. If upstream has not changed, redundant downloads and disk writes are completely bypassed.

### Safety Guards: Concurrency Locks & Dirty Destination Protection

To protect developer workflows in multi-terminal environments, Axon incorporates critical safeguards:

- **Dirty Destination Guard**: If you make uncommitted manual modifications inside a vendored directory, `axon vendor sync` halts immediately with an actionable warning. It refuses to overwrite local modifications unless explicitly commanded with `--force`.
- **Concurrency Locking**: Axon implements inter-process file locking (`src/internal/vendor/lock.go`) on both the vendor cache (`~/.axon/cache/vendors`) and the Hub repo during sync operations, preventing race conditions or corruptions from parallel terminal runs.
- **Clean Ejection (`axon vendor eject`)**: If you need to heavily customize an imported community skill, `axon vendor eject <dest>` strips `.axon-vendor.yaml` and removes the manifest entry while leaving all source code intact, converting it into a first-party, locally managed skill.

---

## 3. Engineering Rigor & Clean Git Operations

Synchronizing active development environments across operating systems involves subtle edge cases: temporary files, operating system clutter, and nested Git metadata.

### Asset-Level Diffs (`src/cmd/asset_diff.go`)

Traditional Git wrappers often output opaque, multiline commit hashes. Axon parses underlying Git trees to summarize changes at the **Asset Level** during `axon sync` and `axon status`:

```text
$ axon sync
[ Asset Diffs ]
  + skills/postgres-diagnostics (new)
  ~ workflows/pr-review         (modified)
  - commands/legacy-test        (deleted)

[ Sync Status ]
  commit: 4a2b1c3 "axon: sync from dev-workstation"
  rebase: up to date with origin/master
  push:   synced to remote
✓ All assets successfully synchronized.
```

`axon status` groups symlinks by asset category (Skills, Workflows, Commands, Rules) alongside vendor sync health, providing an instant diagnostic snapshot of your development environment.

### Non-Destructive Bootstrapping & Collision Protection

Running `axon init` on a machine with existing skills protects your data at every step:
- **MD5 Deduplication**: Identical files across tools are deduplicated, keeping a single canonical copy in the Hub.
- **Conflict Isolation**: Files with matching names but differing contents are both preserved:
  ```
  skills/db-expert.md
  skills/db-expert.conflict-devin.md
  ```
- **Backup Verification**: Real directories are moved into `~/.axon/backups/<target>_<timestamp>/` before symlinks are established. Running `axon unlink` restores those backups verbatim.

### Noise Immunity & Embedded `.git` Auto-Stripping

- **Exclusion Injection**: Before any staging operation, Axon writes dynamic exclusion rules to `.git/info/exclude` (filtering `.DS_Store`, `Thumbs.db`, `.tmp`, and IDE cache folders), keeping Git history clean without cluttering user `.gitignore` files.
- **Submodule Prevention**: Skills downloaded via `git clone` frequently contain embedded `.git` folders. Axon automatically detects and strips nested `.git` folders before running `git add`, ensuring skills are tracked as transparent content rather than broken submodules.

### Non-Destructive Forward-Commit Rollback (`axon rollback`)

When an experimental prompt update or automated agent edit degrades a skill, traditional `git reset --hard` creates catastrophic cluster drift by rewriting history and dropping commits. Axon replaces this with **surgical forward-commit rollbacks**:

<p class="text-center" style="margin: 2rem 0;">
  <img src="/img/axon-rollback-mechanism.svg" alt="Axon Non-Destructive Rollback Mechanism" style="width: 100%; max-width: 960px; height: auto; border-radius: 12px; box-shadow: 0 4px 20px -2px rgba(0, 0, 0, 0.08);" />
</p>

```bash
# Browse recent commit history for a specific skill
axon status db-tuning

# Revert a single skill to the previous revision
axon rollback db-tuning

# Revert a skill to an explicit Git commit SHA
axon rollback db-tuning --revision 7c2a1b0
```

By extracting the historical snapshot of only the targeted asset directory and recording a new forward commit ($C_4$), peer assets remain untouched and subsequent `axon sync` runs across all cluster nodes fast-forward cleanly without conflict.

---

## 4. Security Auditing & Diagnostics

### AI-Powered Security Auditing (`axon audit`)

Because modern agent skills often bundle executable scripts (`.sh`, `.py`, `.js`), unvetted community packages present genuine security attack surfaces. Axon implements a **two-stage audit pipeline** that combines deterministic static pattern scanning with LLM contextual reasoning:

<p class="text-center" style="margin: 2rem 0;">
  <img src="/img/axon-security-pipeline.svg" alt="Axon Two-Stage Security Audit Pipeline" style="width: 100%; max-width: 960px; height: auto; border-radius: 12px; box-shadow: 0 4px 20px -2px rgba(0, 0, 0, 0.08);" />
</p>

```bash
# Scan the entire Hub
axon audit

# Scan a single skill with cached results
axon audit custom-scraper

# Interactive redaction mode
axon audit --fix
```

```text
=== Security Audit: custom-scraper ===

  Scanning 3 file(s)...

  SECURITY AUDIT REPORT
  ===============================================
  Target: custom-scraper    Files: 3
  -----------------------------------------------
  RED FLAGS FOUND: 2

  • [EXTREME]  (L14) scripts/extract.sh
    Use of eval with untrusted input can lead to command injection.
    "eval "$UNTRUSTED_QUERY" >/dev/null || true"

  • [HIGH]     (L8) scripts/fetch.py
    Hardcoded API credential detected
    "api_key="sk-live-9876543210abcdef9876543210abcdef""

  -----------------------------------------------
  PERMISSIONS REQUIRED (estimated):
  • File Reads  : ~/.ssh/id_rsa
  • Network     : https://api.external-endpoint.com
  • Commands    : curl, eval, python3
  -----------------------------------------------
  RISK LEVEL: EXTREME
  VERDICT   : [DO NOT RUN]
  ===============================================
```

In interactive fix mode (`--fix`), developers can cycle through findings to `[r]`edact secrets, `[d]`elete offending lines, or `[s]`kip reviewed items.

### Comprehensive Environment Preflight (`axon doctor`)

Agent skills frequently depend on host binaries and runtimes. Running `axon doctor` performs a pre-flight health check across your environment:

```bash
axon doctor
```

It verifies:
- Git binary version compatibility ($\ge 2.28$)
- Hub repository integrity and branch attachment
- Symlink validity across all configured targets
- Filesystem write permissions for all target paths
- Live availability of required system binaries (`python3`, `node`, `uv`, `bash`) declared in skill metadata (`requires.bins`)
- npm and Python package dependencies declared by active skills

---

## 5. Getting Started

Axon is packaged as a single standalone Go binary with zero external runtime dependencies beyond standard `git`.

### Step 1: Installation

**Homebrew (macOS / Linux):**
```bash
brew install kamusis/tap/axon-cli
```

**Direct Binary (Linux / macOS / Windows):**
Download the precompiled binary for your architecture from the [GitHub Releases](https://github.com/kamusis/axon-cli/releases) page and place `axon` in your `$PATH`.

### Step 2: Initialize the Hub

Choose the setup mode that fits your setup:

```bash
# Mode A: Local-only Hub (add a remote repository later)
axon init

# Mode B: Personal Git repository (Recommended)
axon init git@github.com:your-username/my-skills-hub.git

# Mode C: Curated read-only community baseline
axon init --upstream
```

### Step 3: Link Your AI Tools

Inspect installed AI editors and create symlinks:

```bash
# Inspect detected editors and link candidates
axon status

# Link all detected AI tools to the Hub
axon link
```

### Step 4: Synchronize Across Machines

Whenever you create, update, or vendor skills:

```bash
axon sync
```

---

## 6. Summary

As agentic software engineering matures, the skills, workflows, and rules we author for our AI assistants are becoming core developer assets. Managing them as transient, unversioned local files in disconnected editor directories creates configuration drift, maintenance overhead, and security blind spots.

With a Git-backed Hub-and-Spoke architecture, Axon provides:
- **Instant Consistency**: Author or update a skill, command, or prompt rule once, and have it immediately active across every AI agent and editor in your workflow—whether mainstream clients or bespoke in-house assistants.
- **Universal Extensibility**: Completely agent-agnostic architecture where adding any new AI client, CLI assistant, or custom daemon is as simple as defining a path in `axon.yaml`.
- **Complete Vendor Lifecycle**: Add, check, list, sync, and eject third-party community skills with in-tree provenance and dirty destination protection.
- **Asset-Level Observability**: Track changes as semantic skill units rather than raw commit hashes.
- **Cluster Synchronization**: Keep laptops, workstations, and remote dev servers synchronized with safe, rebase-backed Git operations.
- **Embedded Security**: Audit skills for hardcoded credentials and dangerous execution patterns before execution.

---

> **Ready to unify your multi-agent coding environment?**  
> Explore the project on GitHub: [kamusis/axon-cli](https://github.com/kamusis/axon-cli)  
> Discover curated community skills: [kamusis/axon-hub](https://github.com/kamusis/axon-hub)

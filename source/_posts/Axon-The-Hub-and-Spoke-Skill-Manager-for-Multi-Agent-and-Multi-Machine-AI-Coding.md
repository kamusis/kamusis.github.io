---
title: >-
  Axon: The Hub-and-Spoke Skill Manager for Multi-Agent and Multi-Machine AI
  Coding
date: 2026-10-11 20:55:00
tags: [AI, Agent, Skill, CLI, Developer-Tools, Architecture, Claude, Windsurf, Antigravity, Open-Source]
---

As developers incorporate AI into their daily workflows, their toolchains rarely converge on a single client. An engineer might rely on **Claude Code** for fast terminal-driven tasks, **Windsurf** or **Cursor** for multi-file code editing, **Google Gemini / Antigravity** for deep architectural analysis, and specialized CLI agents for database administration or infrastructure tasks.

Furthermore, development environments are inherently distributed. A typical workflow spans a MacBook for travel, a high-performance workstation for local builds, and remote Linux servers or WSL for production-like verification.

While using multiple AI coding tools boosts flexibility, it introduces a severe operational challenge: **configuration fragmentation and skill drift**.

Every AI coding environment defines its own proprietary configuration directory:
- Claude Code reads from `~/.claude/skills/`, `~/.claude/commands/`, and `~/.claude/CLAUDE.md`
- Windsurf reads from `~/.codeium/windsurf/skills/`, `global_workflows/`, and `memories/global_rules.md`
- Antigravity and Gemini CLI read from `~/.gemini/config/skills/` and `~/.gemini/GEMINI.md`
- OpenAI Codex reads from `~/.codex/skills/` and `~/.codex/AGENTS.md`
- Roo Code, Trae, Qoder, and Kiro maintain similar bespoke directories

When you refine a complex debugging skill or prompt rule in one editor, those improvements do not propagate to your other tools or machines. Developers often resort to manual copy-pasting, ad-hoc shell scripts, or cumbersome dotfile managers. Over time, prompt definitions diverge, external community skills fall out of date, and sensitive API keys risk leaking into unversioned local files.

To resolve this fragmentation, we built **[Axon](https://github.com/kamusis/axon-cli)**—an open-source, Git-backed, hub-and-spoke skill manager written in Go.

---

## 1. Architectural Philosophy: The Hub-and-Spoke Model

Instead of treating each AI editor as an isolated island, Axon decouples skill storage from editor consumption through a **Hub-and-Spoke** architecture.

![Axon How It Works](https://files.seeusercontent.com/2026/02/28/rnQ5/vscode_picgo_1772260135208.jpeg)

### Core Components

1. **The Hub (`~/.axon/repo/`)**: A canonical, local Git repository that stores all skills, reusable workflows, slash commands, and global rules in standard formats. It acts as the single source of truth across all tools.
2. **The Spokes (Filesystem Symlinks)**: Rather than duplicating files or running complex background file-watchers, Axon establishes filesystem symlinks from each tool's expected configuration directory directly into subdirectories of the Hub.
3. **The Remote Repository**: A private or public Git repository (e.g., GitHub or GitLab) configured as the Hub's upstream origin, enabling cross-machine synchronization.

```
+-------------------------------------------------------------------------+
|                        Remote Skills Repository                         |
|                     (GitHub / GitLab / Self-Hosted)                     |
+-------------------------------------------------------------------------+
                                     ^
                                     |  axon sync
                                     v
+-------------------------------------------------------------------------+
|                     Axon Hub (~/.axon/repo/)                            |
|                                                                         |
|   skills/               workflows/            commands/    rules/       |
|   ├── db-tuning/        ├── review.md         ├── mop      ├── code.md  |
|   └── web-search/       └── deploy.md         └── git-pr   └── security |
|                                                                         |
|   global_rules.md  <-- Single source for unified agent system rules     |
+-------------------------------------------------------------------------+
       |                     |                     |               |
       | symlink             | symlink             | symlink       | symlink
       v                     v                     v               v
+--------------+     +---------------+     +---------------+     +--------+
| Claude Code  |     |   Windsurf    |     |  Antigravity  |     | Codex  |
| ~/.claude/   |     | ~/.codeium/   |     |   / Gemini    |     |~/.codex|
|  skills/     |     |   windsurf/   |     |  ~/.gemini/   |     | skills/|
|  CLAUDE.md   |     |   skills/     |     |   config/     |     | AGENTS |
+--------------+     +---------------+     +---------------+     +--------+
```

### Why Symlinks Outperform File Sync Daemons

File-syncing background daemons or bidirectional file copiers frequently encounter race conditions, file locking issues, and partial-write corruptions when multiple editors are open simultaneously. 

By utilizing native operating system symlinks:
- **Zero Latency**: When an AI agent modifies or writes a skill in Claude Code, the underlying file in `~/.axon/repo/skills/` updates immediately.
- **Instant Consistency**: Windsurf, Antigravity, and Codex observe the change on their next file read without intermediate synchronization steps.
- **Zero Resource Footprint**: There are no persistent background daemons consuming CPU cycles or memory.

---

## 2. Core Capabilities & Engineering Details

Axon is engineered to handle real-world developer workflows, from initial migration to multi-machine convergence and security enforcement.

### 2.1 Non-Destructive Bootstrapping & Collision Handling

When running on an existing development machine, AI tool directories are often populated with pre-existing, heterogeneous skills. Running `axon init` followed by `axon link` handles this gracefully:

- **MD5 Deduplication**: If multiple editors contain identical copies of a skill, Axon retains a single canonical copy in the Hub.
- **Content-Conflict Preservation**: If two tools contain different files with the same name, Axon preserves both by appending a deterministic collision suffix:
  ```
  skills/db-expert.md
  skills/db-expert.conflict-windsurf.md
  ```
- **Automatic Backup Snapshots**: Before converting any non-empty directory into a symlink, Axon creates a timestamped archive under `~/.axon/backups/<target>_<timestamp>/`. If you ever run `axon unlink`, Axon removes the symlink and restores your original directory verbatim.

### 2.2 Deterministic Multi-Machine Sync (`axon sync`)

Synchronizing Git repositories across active workstations often encounters friction: operating system metadata files (`.DS_Store`, `Thumbs.db`) pollute commits, and community skills installed via `git clone` introduce nested `.git` folders that break repository tracking.

Axon automates these maintenance tasks before every Git operation:
1. **Automated Exclusion Injection**: Axon writes dynamic exclusion patterns to `.git/info/exclude` prior to staging, preventing temporary files, editor caches (`.idea/`, `__pycache__/`), and OS artifacts from entering Git history.
2. **Embedded `.git` Auto-Stripping**: When community skills are cloned directly into the Hub, they often include their own `.git` metadata. Axon automatically strips nested `.git` folders during sync so skill files are tracked cleanly as plain content rather than unresolvable submodules.
3. **Rebase-Based Synchronization**: `axon sync` performs `git add` → `git commit` → `git pull --rebase` → `git push` in `read-write` mode, ensuring a linear and clean commit history across machines.

### 2.3 External Vendor Mirroring without Git Submodules (`axon vendor sync`)

Managing third-party skills from community repositories (such as Anthropic's public skills repository or OpenAI's curated prompts) via Git submodules is notoriously fragile. Submodules complicate recursive clones and introduce detached HEAD states.

Axon introduces **Vendor Mirroring** via Git sparse-checkout:

```yaml
# ~/.axon/axon.yaml
vendors:
  - name: skill-creator
    repo: https://github.com/anthropics/skills.git
    subdir: skills/skill-creator
    dest: skills/skill-creator
    ref: main
  - name: openai-docs
    repo: https://github.com/openai/skills.git
    subdir: skills/.curated/openai-docs
    dest: skills/openai-docs
    ref: main
```

When you execute `axon vendor sync`:
1. Axon clones or updates the external repository in an isolated cache directory (`~/.axon/cache/vendors/`).
2. It executes a sparse-checkout targeting only the declared `subdir`.
3. It mirrors plain files into the Hub destination without any Git metadata.
4. Upstream commit SHAs are tracked locally; if upstream hasn't changed, redundant downloads are skipped.

### 2.4 Forward-Commit Rollback (`axon rollback`)

When an experimental prompt or automated agent modification degrades a skill's behavior, reverting the change shouldn't require complex Git operations.

Axon provides granular rollback for individual skills or the entire Hub:

```bash
# Inspect recent revision history for a specific skill
axon status db-tuning

# Revert a single skill to its previous commit
axon rollback db-tuning

# Revert a skill to a specific Git commit SHA
axon rollback db-tuning --revision a1b2c3d
```

Crucially, `axon rollback` **never rewrites Git history**. It calculates the inverted patch and creates a new forward commit. This design guarantees that subsequent `axon sync` invocations safely propagate the reversion to all other machines without causing divergence or push rejections.

---

## 3. Security Auditing & Environment Diagnostics

As AI skills increasingly incorporate executable shell scripts, Python helpers, and terminal execution directives, unvetted skills pose real security risks.

### 3.1 AI-Powered Security Audit (`axon audit`)

Axon includes a built-in static and LLM-assisted security scanner designed to evaluate skills before they are shared or executed:

```bash
# Scan the entire Hub
axon audit

# Scan a specific skill
axon audit web-search

# Interactive redaction mode
axon audit --fix
```

```text
=== Security Audit: custom-web-scraper ===

  Scanning 3 file(s)...

  SECURITY AUDIT REPORT
  ===============================================
  Target: custom-web-scraper    Files: 3
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

The audit engine scans for:
- **Hardcoded Secrets**: API keys, private certificates, and personal tokens.
- **Command Injection Vectors**: Dangerous calls like `eval`, `sh -c`, and unescaped variable expansions.
- **Data Exfiltration**: Unexpected outbound network requests (`curl`, `nc`, `wget`) targeting sensitive files like `~/.ssh` or environment variables.

In interactive fix mode (`--fix`), developers can review each finding and choose to `[r]`edact with placeholders, `[d]`elete the offending line, or `[s]`kip false positives.

### 3.2 Pre-Flight Diagnostics (`axon doctor`)

Agent skills frequently depend on underlying system binaries (`node`, `python3`, `uv`, `bash`) or specific package modules.

Running `axon doctor` performs a comprehensive pre-flight verification:

```bash
axon doctor
```

It validates:
- Git binary version and configuration health
- Hub repository integrity and branch tracking
- Symlink validity across all 20+ supported AI tools
- Filesystem write permissions for all target directories
- Live availability of required system binaries declared in skill metadata (`requires.bins`)
- Node.js and Python runtime packages declared by active skills

If a dependency is missing, Axon pinpoints the exact skill requiring it and provides the corresponding installation command.

---

## 4. Quick Start: Getting Started with Axon

Axon is packaged as a single standalone Go binary with zero external runtime dependencies beyond standard `git`.

### Step 1: Install Axon

**Homebrew (macOS / Linux):**
```bash
brew install kamusis/tap/axon-cli
```

**Direct Binary (Linux / macOS / Windows):**
Download the precompiled binary for your architecture from the [GitHub Releases](https://github.com/kamusis/axon-cli/releases) page and place it in your `$PATH` (e.g., `/usr/local/bin` or `~/.local/bin`).

### Step 2: Initialize Your Hub

Choose between three initialization modes:

```bash
# Mode A: Initialize a local-only Hub (add a remote repository later)
axon init

# Mode B: Initialize and link to your personal skills Git repository (Recommended)
axon init git@github.com:your-username/my-skills-hub.git

# Mode C: Initialize as a read-only consumer of the public community Hub
axon init --upstream
```

### Step 3: Link Your AI Tools

Inspect your installed AI editors and create symlinks:

```bash
# Inspect installed tools and candidates
axon status

# Link all detected AI tools to the Hub
axon link
```

### Step 4: Synchronize

Whenever you create or edit skills, commit and push changes across all your machines with:

```bash
axon sync
```

---

## 5. Summary & Roadmap

As agentic software engineering evolves, the skills, rules, and workflows we write for our AI assistants are becoming core intellectual property. Treating them as transient, unversioned local files in disparate editor folders leads to friction, security vulnerabilities, and context drift.

By unifying configuration around a Git-backed Hub-and-Spoke architecture, Axon allows developers to:
- Write a skill once and make it instantly available in **Claude Code, Windsurf, Cursor, Antigravity, Codex, Roo Code, and Qoder**.
- Maintain unified system instructions (`global_rules.md`) across every tool.
- Keep laptops, workstations, and remote development clusters in continuous synchronization.
- Audit external and internal skills for security risks before execution.

---

> **Ready to streamline your multi-agent coding setup?**  
> Explore the project on GitHub: [kamusis/axon-cli](https://github.com/kamusis/axon-cli)  
> Discover curated, community-tested skills: [kamusis/axon-hub](https://github.com/kamusis/axon-hub)

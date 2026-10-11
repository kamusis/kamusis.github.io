---
title: >-
  Bringing the Legendary "grill-me" Skill into Mopheus: Teaching AI Agents Self-Questioning and Adversarial Review
date: 2026-09-07 03:55:00
tags: [Mopheus, AI-Agents, Code-Review, Software-Engineering, Architecture]
---

# Bringing the Legendary "grill-me" Skill into Mopheus: Teaching AI Agents Self-Questioning and Adversarial Review

> *The code is written, the architecture plan is signed off—but do you genuinely feel confident?*  
> *When massive traffic spikes hit, network blips occur, or edge-case anomalies emerge, where will your system fail first?*  
> *We rarely lack hands to write code; what we sorely lack is a rigorous, demanding, and uncompromising "reviewer."*

![Bringing the Legendary "grill-me" Skill into Mopheus](https://files.seeusercontent.com/2026/08/29/O8mb/grill_me_cover_1787996414817.jpg)

---

## 1. What is grill-me? Why is a Bare 60-Word Prompt Hailed as Legendary?

Among developers practicing AI-assisted coding and systems design, one skill is widely revered as a secret weapon: **`grill-me`** (literally meaning to put someone on the hot grill).

Created and open-sourced by renowned engineer and developer educator **Matt Pocock**:
* **Original GitHub Repository**: [mattpocock/skills](https://github.com/mattpocock/skills)
* **Original Skill Reference**: [`skills/grill-me/SKILL.md`](https://github.com/mattpocock/skills/tree/main/skills/grill-me)

This developer-favorite skill has an astonishingly concise definition file (`SKILL.md`)—just three sentences, roughly 60 English words:

```markdown
Interview me relentlessly about every aspect of this plan until we reach a shared understanding. 
Walk down each branch of the design tree, resolving dependencies between decisions one-by-one. 
You MUST provide your recommended answer for every single question — never ask a question without also giving your own recommendation. This is mandatory, not optional.

Ask the questions one at a time.

If a question can be answered by exploring the codebase, explore the codebase instead.
```

---

### Why is Such a Brief Prompt So Uncannily Powerful?

Those new to Prompt Engineering often assume that greater power demands labyrinthine, multi-page instructions. `grill-me` represents the pinnacle of minimalist design:

1. **"Relentlessly until shared understanding"**:  
   Fundamentally dismantles LLMs' default tendency toward sycophancy and superficial praise, instantly anchoring the Agent in the persona of a demanding Principal Architect or rigorous technical interviewer.
2. **"Walk down each branch of the design tree"**:  
   Forces systematic depth-first search across decision paths, knocking down architectural dependencies one by one rather than jumping erratically between surface-level topics.
3. **The Core Anchor: "MUST provide your recommended answer"**:  
   This is the stroke of genius in the prompt. If an AI merely asks questions, it easily devolves into an annoying contrarian. But by **mandating that every critique must come paired with a concrete recommendation**, the model is forced to simulate trade-offs and evaluate alternatives simultaneously. This drastically lowers cognitive overhead for the person being grilled and instantly establishes a high-quality reference baseline.
4. **"Ask one at a time" & "Explore the codebase instead"**:  
   The former prevents cognitive overload by avoiding daunting walls of text; the latter keeps the AI pragmatic and grounded, ensuring it never wastes human time on facts already verifiable in code.

---

## 2. Why Forge grill-me into a Native Ticket Mechanism in Mopheus?

Mopheus natively supports Agent Skills. So why go through the effort of engineering a dedicated native Griller inquiry and review mechanism right within ticket comment threads?

### The Reality of Human-Agent Grilling: Invaluable, Yet Cognitively Exhausting
The original `grill-me` fundamentally operates as **"Agent grilling Human."** An engineer sits at their keyboard, engaging in a live, back-and-forth dialogue with the Agent.

A standard `grill-me` session typically spans **10 to 20 rounds of probing inquiries**, taking roughly **30 minutes**.

In practice, this is awe-inspiring yet mentally intense: **when questioned on the 10th consecutive sharp edge case, one genuinely begins to feel the heat of the grill.**

Yet there is no question: this process is **immensely worthwhile**—especially for core feature designs, complex online database migrations, or distributed systems architecture, where catching catastrophic vulnerabilities before writing code saves days or weeks of firefighting later.

### The Dilemma: How to Run a "Faster Round" Without Sacrificing Depth?
Crucial as deep review is, we cannot afford to lower standards simply because the process is demanding. In fast-paced daily development, teams often need to **run a rapid Grill cycle**, ruthlessly uncovering hidden risks in designs and code in minimal time.

Moreover, in an AI-Native engineering organization, much of the design proposals and pull requests are already written by execution Agents. Having the same Agent review its own work introduces obvious confirmation bias. Meanwhile, waiting for human engineers to schedule a 30-minute synchronous interview for every PR or design detail creates severe delivery bottlenecks.

We needed rigorous reviews to be **automated, high-frequency, and continuous**.

This sparked an intuitive realization: **What if we assign an independent, specialized critique Agent to "interview itself" in the background across 20 rounds of deep reasoning?**

### The Core Solution: Autonomous Self-Questioning and Adversarial Review

By dispatching dedicated specialist Agents (e.g., DBA Guardians, Security Auditors, High-Concurrency Architects) as visiting experts, Mopheus conducts **autonomous multi-round Self-Q&A sessions** directly against any proposal or discussion thread.

```text
Workflow Evolution in Mopheus:

[Deep Human-Agent Duels (Fully Supported)]
Human presents design ──> Agent conducts ruthless Q&A ──> Human responds & iterates ──> (Ideal for major architecture designs)

[Mopheus Native Griller Loop]
Any proposal or discussion thread on a Ticket
       │
       ▼ Click "Grill this comment"
Summon a dedicated Griller Agent (e.g., DBA Guardian / Security Auditor)
       │
       ▼ Background Sandbox (executes multi-round adversarial Self-Q&A)
Griller simulates edge cases ──> Inspects code & constraints ──> Self-questions & uncovers gaps ──> Compiles evidence
       │
       ▼ Automated Response
Publishes a structured, multi-tier review report directly as a threaded reply!
```

This means: **A human engineer simply clicks a button on a ticket, and an agent conducts the entire adversarial review in a background sandbox within minutes, presenting a structured stress-test report right under the thread for rapid team consensus.**

![image-20260823123100801](https://files.seeusercontent.com/2026/08/23/qs5O/image-20260823123100801.png)

---

## 3. What If Humans Still Want to Be Grilled Personally? Absolutely Supported!

Making Griller a native ticket mechanism in no way detracts from the value or thrill of human engineers undergoing the challenge themselves.

If you are a Tech Lead or Principal Architect designing a mission-critical system and want to experience AI's trial by fire directly, doing so in Mopheus is straightforward:

1. **Import or Create Classic Skills**: Create or import the original `grill-me` in the Mopheus Skill Hub, or import the advanced **`grill-with-docs`** (which grills your design while incrementally updating the project's `CONTEXT.md` glossary and `ADR` Architecture Decision Records based on reaching consensus);
2. **Engage in Live Dialogue**:
   - **In Chat**: Start an interactive session with an architect agent and invoke `/grill-me` for real-time stress testing;
   - **In Tickets**: Open an architecture design ticket, @mention your agent in a comment with instructions, and engage in public, round-by-round review within the thread.

```text
Dual-Track Review System:
┌────────────────────────────────────────────────────────────────────────┐
│ 1. Asynchronous Multi-Agent Grilling (Native Griller)                  │
│    Automated Self-Q&A reports on tickets & PRs, exposing hidden flaws  │
├────────────────────────────────────────────────────────────────────────┤
│ 2. Deep Human-in-the-Loop Grilling (grill-me / grill-with-docs Skills)  │
│    Synchronous, multi-round duels elevating engineering design thinking│
└────────────────────────────────────────────────────────────────────────┘
```

---

## 4. How We Built It: Clean, Elegant, and Pragmatic Engineering

To seamlessly weave rigorous grilling into daily workflows, we adhered to disciplined, minimalist engineering principles:

### 1. Agents as Grillers: Non-Invasive Role Specialization
We avoided bloating the system with complex configuration layers. Enabling `is_griller` in an agent's profile (or passing `--is-griller` via the CLI) immediately qualifies that agent as a Reviewer.

The agent naturally leverages its domain specialization:
* **DBA Griller**: Unpacks slow queries, deadlocks, missing indexes, and unsegmented transaction risks;
* **Architecture Griller**: Audits service decoupling, fallback circuits, and state machine transitions;
* **Frontend Griller**: Tests responsive boundaries, accessibility, and UI race conditions.

Regardless of whether a ticket is assigned to a human or another Agent, users can summon them on demand as "visiting experts" **without disrupting the ticket's primary ownership**.

### 2. Precise Prompt Scoping and Closed-Loop Threading
To ensure agents remain grounded and avoid hallucinations during self-Q&A, we instituted three fundamental prompt rules:
* **Thread Scoping**: The Griller strictly consumes the targeted comment and its immediate conversational context as ground truth;
* **Language Matching**: If the source discussion is in Chinese, the entire self-Q&A and final critique are generated in Chinese; if in English, it outputs English, preserving conversational consistency;
* **Thread Reply**: The finished report automatically mounts as a nested child response (`--parent`) beneath the original comment, matching GitHub PR review cleanliness.

### 3. CLI-First Design and Real-Time Awareness
Developers can summon a Griller with one click in the Web UI:

![Mopheus Web One-Click Griller Review](https://files.seeusercontent.com/2026/08/22/b3rI/aidevops_core.png)

Or directly trigger reviews from terminal CLIs or CI/CD pipelines:
```bash
# Launch a deep Grill review on a comment with a designated DBA Griller
mopheus ticket comment grill <comment-id> --agent-id <griller-id>
```

Backed by real-time WebSocket event dispatching and browser tab wake-up reconnects, review tasks update seamlessly whether an engineer stays focused on the tab or returns later.

---

## 5. Future Roadmap: From Single-Agent Self-Q&A to the "Agent Arena Topic"

Autonomous Self-Q&A in isolated sandboxes has already delivered substantial quality gains, but it is only the prelude to true multi-agent adversarial collaboration.

Our next milestone is **enabling two fully independent agents to engage in real-time, face-to-face cross-examination**:
* Dispatching a dedicated Reviewer Agent to directly grill a **Developer Agent preparing to write code**;
* Directing Security and SRE Reviewer Agents to cross-examine a **DBA Agent planning high-risk data migrations or online maintenance**.

To realize this vision, we are designing the **Agent Arena** directly atop Mopheus's existing **Topic** deliberation framework:

```text
Future Roadmap: Agent Arena Topics

Complex Proposal / High-Risk Operation in Ticket Comments
       │
       ▼ One-Click: "Create Arena Topic"
System spawns a dedicated deliberation arena, bringing both Agents into the ring:
┌─────────────────────────────────────────────────────────────┐
│  Mopheus Arena Topic Deliberation Space                     │
│                                                             │
│  [Execution Agent: Feature Builder]  VS  [Review Agent: DBA Griller] │
│                                                             │
│  1. Reviewer: "How will cross-shard aggregation latency be  │
│     guaranteed in this table partitioning strategy?"        │
│  2. Builder:  "Secondary index lookup with distributed      │
│     Bloom filters."                                         │
│  3. Reviewer: "How do you recover if Bloom filter false     │
│     positives surge under network partitions?"              │
│  4. Builder:  "Fallback read-only circuit with revised      │
│     state machine transitions."                             │
│                                                             │
│  ── Through rigorous dialectic debate, both agree on an      │
│     indisputably optimal architecture ──                    │
└──────────────────────────────┬──────────────────────────────┘
                               │
                               ▼ Automated Sync
Conclusions synthesize into a high-confidence decision report, written back to the ticket for safe execution!
```

By elevating ticket discussions into Arena Topics, human engineers can observe high-level algorithmic debate in real time while securing thoroughly stress-tested, resilient architectural designs.

---

## 6. Conclusion: Toward Software Engineering Driven by AI Adversarial Evolution

As LLMs generate code with breathtaking speed, the true bulwark of software engineering shifts toward **review and verification**.

Bringing `grill-me` into Mopheus represents a pivotal milestone in **multi-agent adversarial collaboration**:
* It preserves the training ground where humans can personally undergo AI's rigorous cross-examination;
* It introduces automated quality checkpoints where agents asynchronously self-question and poke holes in human and AI proposals alike;
* And it sets the stage for genuine dual-agent sparring inside Arena Topics.

---

> **Ready to empower your engineering workflows with relentless, professional Griller agents?**  
> Explore [Mopheus (mopheus.ai)](https://www.mopheus.ai) and equip your workspace with dedicated architecture and quality guardians today!

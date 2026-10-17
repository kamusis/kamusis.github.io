---
title: 'A Twelve-Year-Old Oracle Mystery, REA, and the End of the Software License Era'
date: 2026-10-17 17:50:00
tags: [Oracle, ASMFD, Linux, Kernel, REA, Reverse-Engineering, LLM, Gemini, AI]
---

![A Twelve-Year-Old Oracle Mystery, REA, and the End of the Software License Era](https://files.seeusercontent.com/2026/10/17/bXc0/asmfd_rea_cover_png_179221361247.jpg)

On November 20, 2014, shortly after Oracle Database 12.1.0.2 was released, I published a hands-on blog post testing one of its headline storage features: *[Oracle ASM Filter Driver (ASMFD) – New Features for Oracle ASM 12.1.0.2](https://www.dbform.com/2014/11/20/oracle-asm-filter-driver-new-features-for-oracle-asm-12.1.0.2-2/)*.

The context back then was straightforward. When deploying Oracle RAC on Linux, DBAs had to prevent block device names (`/dev/sdb`, `/dev/sdc`, etc.) from drifting across reboots—either by installing ASMLIB or by maintaining brittle `udev` rules. With 12.1.0.2, Oracle introduced **ASMFD (ASM Filter Driver)**, promising not only to replace both ASMLIB and `udev`, but also to deliver a killer feature: **I/O Filtering**. ASMFD claimed it could reject any non-Oracle write I/O to ASM disks—even accidental destructive writes issued by `root`.

To see whether this was marketing hype or reality, I ran a borderline suicidal experiment in my lab.

I provisioned a 200 MB disk `/dev/sdo` with the AFD label `ASMDISK99`, created an ASM diskgroup `DGTEST`, switched to `root`, and executed every DBA's worst nightmare—zeroing out the entire block device with `dd`:

```bash
[root@dbserver1 ~]# dd if=/dev/zero of=/dev/sdo
dd: writing to `/dev/sdo': No space left on device
409601+0 records in
409600+0 records out
209715200 bytes (210 MB) copied, 19.9602 s, 10.5 MB/s
```

What happened next defied everything I thought I knew about Linux storage:

1. **`dd` completed without a single I/O error**, happily writing 200 MB of zeros to the end of the device.
2. When I inspected the raw block device from the OS using `od -c -N 256 /dev/sdo` and `strings /dev/sdo`, **every byte came back as `\0`—the ASM disk header was completely gone**.
3. Yet inside Oracle, the `DGTEST` diskgroup could still be dismounted and mounted cleanly. Creating tablespaces, creating tables, inserting rows, and forcing checkpoints all worked without a hitch. Reading the disk via `kfed read AFD:ASMDISK99` showed the disk header metadata completely intact.
4. Most baffling of all: **even after I restarted the ASM instance and restarted the entire CRS clusterware stack, running `od` on `/dev/sdo` from the OS still returned all zeros! Only after I rebooted the entire Linux server did `od -c -N 256 /dev/sdo` suddenly show the original `ORCLDISKASMDISK99` header again, as if it had risen from the dead.**

At the time, I couldn't explain how the same `/dev/sdo` could look zeroed out to the OS across cluster restarts yet magically recover upon a host reboot. I ended my 2014 post with an open question:

> *"Until the server itself was rebooted (restarting ASM or restarting Clusterware still showed zeroed-out data at the OS level), all the data came back. I am still not sure what kind of redirection technology Oracle used to achieve this magical effect."*

That question sat unanswered on my blog for nearly twelve years.

---

## 1. Solved in Seconds Twelve Years Later: When Gemini 4 Argon Doesn't Need a Single Binary

This morning, while experimenting on my MacBook with **[REA (Reverse Engineer Anything)](https://github.com/morluto/rea)**—the reverse-engineering MCP server that has been trending across GitHub over the past two weeks—I started wondering how far AI-driven binary analysis could go against massive proprietary enterprise software like Oracle Database.

That train of thought reminded me of my twelve-year-old ASMFD mystery. Obviously, my MacBook doesn't have Oracle 12c installed, nor do I have the `oracleafd.ko` kernel module or `libafd12.so` shared library lying around.

Out of curiosity, I dropped the link to my 2014 blog post into **Gemini 4 Argon** and asked: *"I don't have Oracle on this machine right now—can you deduce the underlying mechanism behind that bizarre phenomenon first?"*

Within seconds, Gemini 4 Argon gave me an answer that sent chills down my spine. Without seeing a single line of disassembled binary code, relying solely on three terminal snippets from my 2014 post, it reconstructed the exact kernel and storage call path down to the struct level:

1. **Clue #1**: In my post, `ls -l /dev/oracleafd/disks/ASMDISK01` showed a file size of just **9 bytes** (`-rw-r--r-- 1 root root 9`). It wasn't a block device node at all—it was a plain text file containing the string `/dev/sdo\n`.
2. **Clue #2**: The ASM alert log contained `Loaded library: /opt/oracle/extapi/64/asm/orcl/1/libafd12.so` and probed the character device `/dev/oracleafd/admin` during initialization.
3. **Clue #3**: While `dd` was running, `/var/log/messages` logged a kernel function warning:
   ```text
   afd_mkrequest_fn: write IO on ASM managed device (major=8/minor=224) not supported
   ```

Gemini 4 Argon went straight to the root cause: **There was never any "write redirection technology." What I witnessed in 2014 was an optical illusion jointly created by a Linux Block Layer request-queue hook and the VFS Page Cache.**

Here is the exact mechanism it laid out:

![Oracle ASMFD Block Layer Hook and Page Cache Illusion](https://files.seeusercontent.com/2026/10/17/l0Db/asmfd_io_flow_en.png)

* **Step 1 (The Kernel Block Layer Hook)**: When `oracleafd.ko` takes over `/dev/sdo`, it hijacks the `make_request_fn` function pointer on the Linux block device's `struct request_queue`, replacing it with `afd_mkrequest_fn`. Legitimate Oracle processes (and `kfed`) load `libafd12.so`, communicate through `/dev/oracleafd/admin`, and submit **Direct I/O** requests carrying a trusted kernel tag. Any untagged write `bio` sent directly to `/dev/sdo`—even by `root` running `dd`—is intercepted and dropped inside `afd_mkrequest_fn` **before it ever reaches the underlying SCSI driver**. The physical disk sectors were never touched.
* **Step 2 (Why `dd` Reported Success and `od` Read Back All Zeros)**: Look at the command I ran in 2014: `dd if=/dev/zero of=/dev/sdo`—**without `oflag=direct`**!
  On Linux, writing to a block device without `O_DIRECT` goes through **Buffered I/O**. `dd` merely wrote `0x00` into the in-memory **Block Device Page Cache** for `/dev/sdo`, marked those RAM pages `PG_uptodate` and `PG_dirty`, and returned success to user space immediately.
  Milliseconds later, when the kernel background `flusher` thread attempted to write those dirty pages to disk, `afd_mkrequest_fn` rejected the write `bio`s with `-EIO`. Crucially, **when background writeback fails on Linux, the kernel clears `PG_dirty` and logs an I/O error, but it does NOT invalidate or evict the in-memory page—the `PG_uptodate` flag remains set in RAM!**
* **Step 3 (Why Restarting Clusterware Did Nothing, but Rebooting the OS Brought Everything Back)**:
  When I subsequently ran `od` and `strings` against `/dev/sdo`, they also performed Buffered Reads. Seeing `PG_uptodate` pages in the `/dev/sdo` Page Cache, the Linux kernel happily served the in-memory zeros straight from RAM without ever reading the physical disk. Restarting ASM or Oracle Clusterware only restarts user-space daemons; the kernel's block device Page Cache remains untouched.
  Meanwhile, Oracle ASM, database processes, and `kfed` always open disks with `O_DIRECT`, bypassing the OS Page Cache entirely and reading the intact physical sectors on disk. Only when I **rebooted the entire Linux server** was the polluted RAM Page Cache wiped clean—forcing `od` to read from the physical disk again, where `ORCLDISKASMDISK99` had been sitting safely all along.

In other words, **ASMFD achieved its design goal 100%**. Because `oracleafd.ko` stands guard at the **Linux Block Layer**—the choke point below VFS/Page Cache and above the physical SCSI driver—every write path (`dd`, `mkfs`, `fdisk`, `parted`) must eventually become a `struct bio` passing through `afd_mkrequest_fn`. Had I run `dd if=/dev/zero of=/dev/sdo oflag=direct` back in 2014, `dd` would have bypassed the Page Cache, hit `afd_mkrequest_fn` synchronously on the very first block, and failed immediately with `Input/output error`. And had I wanted to see the real disk contents without rebooting the server, a simple `blockdev --flushbufs /dev/sdo` or `echo 3 > /proc/sys/vm/drop_caches` would have evicted the polluted cache on the spot.

A mystery that lingered for twelve years was dissected in seconds—without a single binary file—because frontier reasoning models have internalized the state machines of the Linux kernel and enterprise storage stacks to a degree that rivals a thirty-year systems veteran.

---

## 2. When Frontier LLMs Meet REA: The End of the "Black-Box Software" Era

If Gemini 4 Argon solving a twelve-year-old kernel mystery without binaries shows how deeply LLMs understand system principles, turning back to **[REA (Reverse Engineer Anything)](https://github.com/morluto/rea)** reveals an even more disruptive reality:

**Even when an AI encounters proprietary logic it has never seen in training data, as long as you have the shipped binary or client bundle, the AI can now pry open the black box and reconstruct the source code itself.**

While setting up REA earlier today, I ran into a real-world example on my own laptop. A desktop AI client (`WorkBuddy.app`) had silently modified five `SKILL.md` files in my local skills repository on startup and dropped a mysterious `.model_invocation_to_override_migration.json` file. To find out why, the agent inspected `/Applications/WorkBuddy.app/Contents/Resources/app.asar` and, in less than five seconds, located and de-minified the exact migration routine into clean JavaScript:

```javascript
var TARGET_VERSION = 1;
var MARKER_FILE = ".model_invocation_to_override_migration.json";

async function runSkillMigration(ctx) {
  let result = { alreadyDone: false, scanned: 0, migrated: 0, skipped: 0, failed: 0 };
  let markerPath = path.join(ctx.userSkillsDir, MARKER_FILE);
  // Short-circuits if version >= 1 without touching any SKILL.md files
  if (((await readMarker(markerPath))?.version ?? 0) >= TARGET_VERSION) {
    return { ...result, alreadyDone: true };
  }
  // ...
}
```

This is why REA has sent shockwaves through the engineering community. Its official tagline pulls no punches:

> **"See a feature you like. Understand how it works, down to the binary level. Ask your agent to investigate it with REA. It can inspect the app without its source code, explain how the feature works, show the evidence, and build a version for your project."**

Historically, reverse engineering was an arcane craft practiced by a tiny priesthood of security researchers staring at IDA Pro and walls of assembly late into the night. REA changes the game by wrapping more than 170 specialized reverse-engineering tools—spanning Electron/JS bundle deconstruction, Mach-O/ELF/PE inspection, Ghidra/Hopper/IDA decompilation bridges, Frida dynamic instrumentation, packet capture, and SQLite state diffing—into a unified MCP server for coding agents. It turns reverse engineering into **Evidence-Backed Reconstruction**:

1. **For Electron, Web, Node.js, .NET, and Java applications**: Shipping client bundles is now practically shipping open source. Main-process architectures, IPC contracts, embedded prompts, local SQLite schemas, and state machines can be extracted as structured evidence and reconstructed 1:1 into clean TypeScript, C#, or Java in minutes.
2. **For native C/C++/Rust/Go binaries and system software**: An agent doesn't need to decompile a 500 MB monolith all at once. For a feature like Oracle ASMFD, the entire mechanism lives inside a few hundred kilobytes of `oracleafd.ko` and `libafd12.so`. Using REA with Ghidra to slice the call graph around `afd_mkrequest_fn`, an agent can recover equivalent C code in minutes.
3. **Closed-loop verification**: REA even includes `build_reconstruction_obligation_ledger` and `verify_reconstruction`—tools that force the agent to audit every control-flow branch and boundary condition in the original binary against the reconstructed code until functional parity is proven.

---

## 3. Three Inevitable Conclusions for the Future of Software

Put frontier reasoning models (like Gemini 4 Argon) together with automated binary reconstruction toolchains (like REA), and three conclusions about the future of the software industry become almost inescapable:

### Conclusion 1: Nearly All Future Software Will Be AI-Generated
When people said "AI will write all code" a year or two ago, they usually pictured autocomplete on steroids or vibe-coding a greenfield web app from a prompt.
Today, AI is no longer limited to **forward generation** from scratch—it has unlocked **reverse absorption**. Decades of human engineering wisdom, edge-case handling, and protocol tricks locked inside closed-source binaries, desktop clients, and kernel drivers are no longer opaque walls. Coding agents can extract verifiable design evidence from existing binaries, understand the underlying mechanics, and synthesize clean, modern, well-tested implementations in any target stack. **The entire corpus of shipped software has become an open textbook for AI.**

### Conclusion 2: The Business of Selling Pure "Feature Licenses" Is Dead
For forty years, the most lucrative business model in enterprise and commercial software was simple: **compile source code into proprietary binaries, and charge per-CPU or per-seat license fees.**

That entire model rested on one assumption: **replicating complex software features required prohibitive R&D time and capital.** Because it would take a competitor 200 engineers and two years of trial and error to clone your product, customers paid the license tax.

When frontier LLMs + REA collapse the cost of reverse engineering and clean-room reconstruction toward zero:
* Any software whose value lives primarily inside a shipped binary, desktop client, or on-premise package loses its implementation secrecy the moment a release hits the wild.
* Within days—or hours—an agent-empowered team or open-source community can deconstruct the workflow and ship a free, feature-equivalent alternative.
* **"We have proprietary code that others cannot figure out how to write" is no longer a defensible pricing moat.** Standalone utility software, thin middleware wrappers, and static feature-license vendors will see their license premiums compressed to zero.

### Conclusion 3: How Will Software Vendors Survive and Compete?

If source code is no longer a moat and compiled binaries no longer hide secrets, where does durable enterprise value live?

The software companies that thrive in the coming decade will build their moats across four dimensions that `npx rea-agents` can never decompile:

#### 1. Proprietary Context and Data Gravity
REA can dissect every binary instruction, UI component, and algorithm in your client app, **but it cannot decompile the customer's live organizational context, historical decision trails, domain knowledge graph, and distilled long-term agent memory.**
Software is evolving from a stateless "feature executor" into a living organizational brain where people, agents, and teams collaborate. You can clone an application's code overnight, but you cannot clone years of accumulated workflow state, permission topology, and contextual memory.

#### 2. Managed Runtimes, Real-Time Network Effects, and Orchestration
A standalone local binary is trivial to replicate because 100% of its value sits on the user's disk.
When software becomes a **real-time orchestration network** connecting human members, heterogeneous AI agents, enterprise communication channels, and elastic cloud sandboxes, the client binary is merely a thin ticket to the network. The true moat lies in multi-tenant state synchronization, hybrid local/cloud runtime scheduling, security boundaries, and live ecosystem integrations.

#### 3. Deterministic Accountability, Trust, and SLAs
Consider Oracle ASMFD again. Today, an engineer with Gemini and REA can understand every line of `oracleafd.ko` in ten minutes and generate an equivalent Linux kernel module to filter block I/O.
**Yet no bank CIO on earth would run their core payment database on a kernel module "vibe-reconstructed by an AI last night."**
When enterprises pay mission-critical infrastructure vendors, they are not paying for a few hundred kilobytes of C code—they are paying for **battle-tested determinism across thousands of hardware combinations, regulatory compliance, and a legally binding SLA that guarantees accountability when things go wrong at 3:00 AM.** AI can replicate code instantly; it cannot replicate institutional trust or legal liability.

#### 4. Product Taste and Velocity of Evolution
Reverse engineering can only ever copy **yesterday's snapshot**.
While copycats are busy using REA to reconstruct last week's release, category leaders with sharp product taste, deep empathy for real user pain points, and AI-native engineering velocity have already shipped the next paradigm.

---

## Epilogue

Twelve years ago, staring at `/dev/sdo` as its bytes vanished into zeros and reappeared after a reboot, I felt as though enterprise software giants possessed a kind of untouchable low-level magic.

Twelve years later, a frontier LLM unraveled that magic in seconds from first principles, and tools like REA have placed the scalpel in every developer's hands. Now that the black box is gone, the era of collecting rent on hidden code is coming to a close—and a far more exciting era, defined by **product taste, data gravity, live orchestration, and real accountability**, has just begun.

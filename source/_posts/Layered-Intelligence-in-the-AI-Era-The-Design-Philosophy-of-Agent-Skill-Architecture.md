---
title: >-
  Layered Intelligence in the AI Era: The Design Philosophy of Agent-Skill
  Architecture
date: 2026-03-11 16:06:12
tags: [AI, Agent, Skill, Architecture, Database, Operations]
---

# Layered Intelligence in the AI Era: The Design Philosophy of Agent-Skill Architecture

## Introduction: A Database Operations Scenario

It's 3:00 AM, and a production database alert goes off.

**The Traditional Approach** (Status Quo):
1. The On-call DBA is woken up by a phone call.
2. VPN connection → Jump Server → Production Server.
3. Manually execute 10+ diagnostic SQL queries.
4. Stare at dense query results in a terminal window.
5. Rely on intuition to identify the problem, perhaps digging through documentation or historical tickets.
6. Organize the data and write a diagnostic report.
7. **Total elapsed time: 30-60 minutes**, while praying for no human error.

**With a Generic AI Assistant** (An incremental improvement):
```bash
> ChatGPT, help me analyze the performance issues on this database.
> [Paste a mountain of SQL query results]

# AI Reply: Based on the information provided, it might be...
# But the AI doesn't know which SQL to execute next.
# It can't connect directly to the database.
# And it can't analyze multiple dimensions in parallel.
```

**The Agent-Skill Architecture** (Our solution):
```bash
> Background: Start yashandb-ops agent, check 'dev-kamus' database space usage.
> Also, start a yashandb-ops agent to check database wait events.

# 2 minutes later, two complete diagnostic reports are presented simultaneously.
# Including: data analysis, root cause diagnosis, optimization suggestions, and executable SQL.
```

This isn't science fiction; it’s a real-world scenario we just verified. Reducing the time from 30 minutes to 2 minutes, from serial to parallel, and from manual judgment to intelligent analysis—behind it all is a meticulously designed **Agent-Skill Layered Architecture**.

## I. The Three Eras of Operations

### 1.1 The Traditional Era: Manual + Scripts + Manually Coded Products
Most enterprises today use mature operations products, yet we still consider this the "Traditional Era." Moving from manual commands to scripts and UIs is progress, but the capability behind these products remains **manually engineered**, not **autonomously understood and invoked by machines**.

Let's look at the evolution: Purely manual → Scripting → Operations Products.

**The DBA's Manual Workflow**:
```bash
# DBA hand-cranked workflow
$ ssh dba@prod-db-server
$ sqlplus / as sysdba

SQL> -- Check tablespaces
SQL> SELECT tablespace_name, ...
     -- 50 lines of SQL

SQL> -- Check wait events
SQL> SELECT wait_class, event, ...
     -- Another 50 lines of SQL
# Then manually organize data in Excel, write reports...
```

**The Scripting Attempt**:
Scripts solved repetition but introduced new problems:
- **Lack of Intelligence**: Rigid execution without the ability to adjust based on results.
- **Maintenance Hell**: SQL logic scattered across various script files.
- **Unreadable Results**: Output is raw text and requires manual interpretation.
- **Zero Interaction**: Cannot drill down based on initial findings.

Platform like `pgAdmin`, `Oracle Enterprise Manager`, or `Prometheus + Grafana` encapsulate scripts into a friendlier UI, but the core remains: experts **hard-code the logic line by line**.

In essence, the traditional paradigm is:
```text
if Metric A > Threshold:
    Alert
elif Metric B is abnormal:
    Show predefined dashboard
else:
    Wait for human judgment
```

These systems handle only what the designer anticipated. They can "display" but not "understand"; "alert" but not "diagnose".

**The Divide**: The difference between the traditional era and the AI era isn't "whether you have automated products," but rather **"is the capability hard-coded, or is it encapsulated into a Skill that an Agent can invoke dynamically?"**

### 1.2 The Generic AI Assistant Era: Intelligent, but Not Professional
ChatGPT and Claude give DBAs hope, but practical issues remain:
- **No Active Execution**: AI acts as "advanced documentation," requiring humans to run SQL and paste results.
- **Context Pollution**: Long query results fill the context window, causing the AI to lose the thread.
- **Lack of Domain Depth**: Generic AI knows "what" things are but lacks specific diagnostic SQLs and expert optimization experience.
- **Serial Execution**: Cannot handle multiple dimensions (space, performance, locks) simultaneously in a single chat thread.

**The RAG Limitation**: Many look to RAG (Retrieval-Augmented Generation) as the final answer. While RAG increases knowledge density, it solves **Knowledge Retrieval**, not **Execution and Orchestration**. RAG is a vital component, but not the ultimate architecture. It moves AI from "generic answers" to "documented answers," but not yet to "task completion."

### 1.3 The Agent-Skill Architecture Era: Professional + Intelligent + Parallel
This is the architecture we are introducing. Here is a comparison:

| Dimension | Traditional Ops | Generic AI Assistant | Agent-Skill Architecture |
| :--- | :--- | :--- | :--- |
| **Execution** | Manual | AI Guided, Human Executed | AI Auto-Executed |
| **Knowledge Depth** | Individual Experience | Generic Knowledge | Domain Expert Level |
| **Parallelism** | None (one pair of hands) | None (Serial chat) | Native Support |
| **Response Speed** | 30-60 minutes | 15-30 minutes | 2-5 minutes |
| **Consistency** | Varies by person | Decent | Fully Consistent |
| **Error Rate** | Medium (Fatigue/Skill) | Medium (Understanding) | Low (Standardized Flow) |

---

## II. The Architecture Dilemma of Generic Assistants
The problem isn't just "intelligence." It's that the monolithic architecture of generic assistants isn't designed for professional task execution. It mixes understanding, reasoning, knowledge calling, and tool use into a single flow. Professional database ops requires multi-step diagnosis and dynamic decision-making that a single chat thread can't reliably handle.

---

## III. Agent-Skill Architecture: Layered Intelligence

Our solution is a three-layer architecture:

![Multi-Agent Orchestration Architecture](https://files.seeusercontent.com/2026/03/11/0jNv/file_86-f59f2f6b-ddd4-44a2-8b2d-.jpg)

### 3.1 Layer 1: The Orchestrator
**Responsibility**: Understand user intent, decompose complex tasks, and route them to the appropriate executors.

```python
class OrchestratorAgent:
    """
    Orchestrator: Responsible for task understanding and distribution.
    Contains no specific implementation, only intelligent routing.
    """
    def route_task(self, user_request):
        # Intent identification
        intent = self.understand_intent(user_request)

        # Task decomposition
        if intent.is_complex():
            subtasks = self.decompose(intent)
            # Parallel distribution
            return self.dispatch_parallel(subtasks)
        else:
            # Single task routing
            return self.dispatch_single(intent)

    def dispatch_parallel(self, subtasks):
        """Start multiple Agents in parallel"""
        agents = []
        for task in subtasks:
            agent = self.create_agent(
                type=task.agent_type,
                prompt=task.description,
                run_in_background=True
            )
            agents.append(agent)
        return agents
```

### 3.2 Layer 2: Specialized Agent
**Responsibility**: Execute tasks within a specific domain by calling the relevant Skills.

```python
class YashanDBOpsAgent:
    """
    YashanDB Ops Agent.
    Analyzes intent and selects the right Skill.
    """
    def __init__(self):
        self.skills = {
            'space': YashanDBSpaceSkill(),
            'wait_events': YashanDBWaitEventsSkill(),
            'locks': YashanDBLocksSkill(),
            'health': YashanDBHealthSkill(),
            # ... more skills
        }

    def execute(self, task_description):
        intent = self.analyze_intent(task_description)
        skill_name = self.select_skill(intent)
        skill = self.skills[skill_name]
        params = self.extract_parameters(task_description)
        return skill.execute(**params)
```

### 3.3 Layer 3: Skill
**Responsibility**: Encapsulate specific professional knowledge and execution logic.

```python
class YashanDBSpaceSkill:
    """
    Database Space Analysis Skill.
    Encapsulates diagnostic logic and expertise.
    """
    def execute(self, db_connection):
        report = SpaceReport()
        # 1. Analyze tablespace usage (Professional SQL queries)
        # 2. Check datafile and autoextend status
        # 3. Analyze large segments (Top N)
        # 4. Detect fragmented tables
        # 5. Generate intelligent recommendations with SQL
        return report
```

---

## IV. Architectural Advantages: Why This Design?

### 4.1 Separation of Concerns
Every layer does one thing. Orchestrator routes, Agent calls, and Skill executes professional diagnosis logic.

### 4.2 Parallel Execution
In real tests:
- Space check: 99s
- Wait event analysis: 110s
- **Parallel Total: 110s** (vs 209s for sequential)

### 4.3 Context Isolation and Depth
Each Skill has its own context window. The "Space Skill" window is not polluted by lock data or general coding knowledge, keeping the AI focused on expert database metrics.

### 4.4 Extensibility and Evolution
Adding new functionality takes three simple steps:
1. Create a new Skill (e.g., `IndexAdvisorSkill`).
2. Register it to the Agent.
3. Update Orchestrator routing (often automatic).

---

## V. AI Era Design Philosophy

### 5.1 From "General" to "Precise"
Traditional software seeks "monolithic" apps. AI era needs modular intelligence.

### 5.2 Conway's Law in AI Architecture
Melvin Conway stated that systems reflect organization structure. In AI:
Expert Mindset → Skill Modules → Agent Boundaries → Orchestrator Routing.
Our system mirrors the DBA's specialized mindset: Space, Performance, Locks.

### 5.3 Unix Philosophy Reborn
1. **Do one thing well** (Specialized skills).
2. **Design for composition** (Agents combine skills).
Every Skill is like a Unix command—focused and reusable.

---

## VI. Real-World Case: Database Diagnosis in Two Minutes

### 6.1 User Input
"Start yashandb-ops agents in background to check space and wait events on dev-kamus."

### 6.2 Orchestrator Processing
It identifies `intent_1` (space) and `intent_2` (wait events) and dispatches them to parallel background agents.

### 6.3 Agent & Skill Execution
`Agent 1` calls the `SpaceSkill`. It queries tablespace usage, file status, large objects, and fragmentation. It then synthesizes these into a structured `SpaceReport`. `Agent 2` does the same for wait events simultaneously.

### 6.4 Result Presentation
110 seconds later, the user receives structured reports with warnings (e.g., SYSTEM tablespace at 80.1%) and actionable SQL recommendations.

---

## VII. Design Principles & Best Practices

### 7.1 Skill Design Principles
1. **Single Responsibility (SRP)**: Avoid "God Skills".
2. **Interface Consistency**:
```python
class BaseSkill(ABC):
    @abstractmethod
    def execute(self, **params) -> Report:
        """Execute and return structured report"""
        pass
```
3. **Knowledge Encapsulation**: Encapsulate **Declarative** (SQL), **Procedural** (Flows), and **Empirical** (Best practices) knowledge.

### 7.2 Agent Design Principles
1. **Domain Focus**: One agent per database type (PostgreSQL vs MySQL).
2. **Statelessness**: Let Skills or external storage manage state.

### 7.3 Orchestrator Design Principles
1. **Intelligent Routing**: Understand dependencies and automate parallelism.
2. **Graceful Fallback**: Downgrade from specialized agents to generic handlers or human escalation if an agent fails.

---

## VIII. Future Outlook: Self-Evolving Systems

### 8.1 Auto-Generation of Skills
The Orchestrator could generate a new `DeadlockDetectionSkill` on the fly by processing documentation and existing lock knowledge.

### 8.2 Collaborative Agent Networks
Agents interacting to share context.

![image](https://files.seeusercontent.com/2026/03/11/wOi9/vscode_picgo_1773216717119.png)

### 8.3 Continuous Knowledge Accumulation
Every execution is a learning opportunity.
```python
class EvolvingSkill:
    def execute(self, **params):
        result = self.core_logic(**params)
        self.log_execution(params, result)
        feedback = self.collect_feedback(result)
        if feedback.is_positive():
            self.knowledge_base.reinforce(params, result)
        return result
```

---

## IX. Conclusion: Architecture as Philosophy

The Agent-Skill architecture is about **Divide and Conquer**, **Professionalism**, and **Compositional Innovation**. It ensures AI is deeper, faster, more isolated, and ready to evolve.

**Tech Stack**: Claude 4.6 (Sonnet) + Agent SDK + yashandb-cli + Python

---
*Interested in AI architecture? Follow my blog at kamusis.me for more deep dives.*

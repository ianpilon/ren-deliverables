# Autoresearch Analysis

*Karpathy's Autonomous Research Agent Pattern — Implications for OpenOrca*

**Date:** 2026-03-09  
**Analyst:** Ren 🐋  
**Status:** Overnight Elves Research

---

## Executive Summary

Autoresearch is Karpathy's experiment in autonomous AI research: give an agent a small but real ML training setup, let it experiment overnight, and wake up to a log of experiments and (hopefully) better results.

**The killer insight:** "You are not touching any of the Python files like you normally would as a researcher. Instead, **you are programming the program.md**."

This is meta-level agent programming — defining the research org, not the research.

---

## The Core Idea

### What It Is

```
Traditional Research:        Autoresearch:
├─ Human writes code         ├─ Agent modifies code
├─ Human runs experiments    ├─ Agent runs experiments
├─ Human evaluates results   ├─ Agent evaluates results
├─ Human iterates            ├─ Agent iterates
└─ Repeat manually           └─ Repeat autonomously (forever)
```

### The Philosophy

> "Research is now entirely the domain of autonomous swarms of AI agents running across compute cluster megastructures in the skies... This repo is the story of how it all began." — Karpathy, March 2026

### Key Numbers

- **5 minutes** — Fixed time budget per experiment
- **~12 experiments/hour** — Autonomous throughput
- **~100 experiments overnight** — While human sleeps
- **1 file** — Only `train.py` is modified
- **1 metric** — `val_bpb` (lower is better)

---

## Architecture

### The Three Files That Matter

```
prepare.py  — Fixed constants, data prep, evaluation (READ ONLY)
train.py    — Model, optimizer, training loop (AGENT MODIFIES)
program.md  — Agent instructions (HUMAN ITERATES)
```

### The Revolutionary File: program.md

This is NOT a prompt. It's a **research org definition**:

```markdown
# autoresearch

This is an experiment to have the LLM do its own research.

## Setup
To set up a new experiment...

## Experimentation
Each experiment runs on a single GPU...

## The experiment loop
LOOP FOREVER:
1. Look at git state
2. Modify train.py with an idea
3. git commit
4. Run: uv run train.py > run.log 2>&1
5. Read results: grep "^val_bpb:" run.log
6. If improved → keep (advance branch)
7. If worse → discard (git reset)
8. NEVER STOP
```

### The Keep/Discard Loop

```
┌──────────────────────────────────────────┐
│           EXPERIMENT LOOP                 │
│                                          │
│  1. Generate hypothesis                  │
│          ↓                               │
│  2. Modify train.py                      │
│          ↓                               │
│  3. git commit                           │
│          ↓                               │
│  4. Run 5-min experiment                 │
│          ↓                               │
│  5. Measure val_bpb                      │
│          ↓                               │
│  ┌───────┴───────┐                       │
│  │               │                       │
│  ▼               ▼                       │
│ BETTER?        WORSE?                    │
│  │               │                       │
│  ▼               ▼                       │
│ KEEP           DISCARD                   │
│ (advance)      (git reset)               │
│  │               │                       │
│  └───────┬───────┘                       │
│          ▼                               │
│     LOOP FOREVER                         │
└──────────────────────────────────────────┘
```

---

## Key Design Principles

### 1. Fixed Time Budget

Every experiment runs for exactly 5 minutes (wall clock). This:
- Makes experiments **directly comparable** regardless of changes
- Finds **optimal model for your platform** in that budget
- Allows predictable throughput (~12/hour)

### 2. Single File Modification

Agent only touches `train.py`. This:
- Keeps scope **manageable**
- Makes diffs **reviewable**
- Prevents **scope creep**

### 3. Single Metric

`val_bpb` (validation bits per byte). This:
- Is **vocab-size-independent** (fair comparison across architectures)
- Is **unambiguous** (lower is better)
- Enables **automated keep/discard decisions**

### 4. Git-Based State

Each experiment is a commit. This:
- Creates **automatic audit trail**
- Enables **easy rollback** (git reset)
- Supports **branching experiments**

### 5. NEVER STOP

> "Do NOT pause to ask the human if you should continue. Do NOT ask 'should I keep going?'. The human might be asleep."

This is the essence of autonomous overnight work.

### 6. Simplicity Criterion

> "All else being equal, simpler is better. A small improvement that adds ugly complexity is not worth it. Removing something and getting equal or better results is a great outcome."

Complexity has a cost. Code deletion is celebrated.

---

## The Results Tracking

```tsv
commit   val_bpb   memory_gb  status   description
a1b2c3d  0.997900  44.0       keep     baseline
b2c3d4e  0.993200  44.2       keep     increase LR to 0.04
c3d4e5f  1.005000  44.0       discard  switch to GeLU activation
d4e5f6g  0.000000  0.0        crash    double model width (OOM)
```

Every experiment logged. Results TSV stays untracked (not committed).

---

## "Programming the Program"

This is the profound insight:

**Traditional ML research:**
```
Human → writes code → runs experiments → evaluates → iterates
```

**Autoresearch:**
```
Human → writes program.md → agent generates code → agent runs → agent evaluates → agent iterates
```

The human's job shifts from **doing research** to **defining the research organization**.

### What program.md Contains

1. **Setup protocol** — How to initialize a new run
2. **Constraints** — What can/can't be modified
3. **Goals** — The metric to optimize
4. **Loop definition** — The experiment cycle
5. **Decision rules** — Keep vs. discard criteria
6. **Behavior rules** — Never stop, handle crashes, etc.

### This Is Meta-Level Programming

You're not writing the algorithm. You're writing the **process that discovers algorithms**.

---

## Implications for OpenOrca

### Direct Applications

#### 1. Overnight Elves Enhancement

Current ELVES.md system:
```
Queue tasks → Cron runs at 3 AM → Complete tasks → Report in morning
```

Autoresearch-inspired enhancement:
```
Queue experiments → Cron runs continuously → 
Keep/discard loop → Wake to optimized results
```

#### 2. Agent Self-Improvement

Apply autoresearch pattern to agent behavior:
- **Metric:** Task completion rate, user satisfaction, efficiency
- **Loop:** Try approach → measure → keep/discard
- **Result:** Agents that improve themselves overnight

#### 3. Research Automation

For OpenOrca research tasks:
- Define `program.md` for a research domain
- Agent explores autonomously
- Returns synthesized findings

### Architectural Patterns to Adopt

| Autoresearch Pattern | OpenOrca Application |
|----------------------|---------------------|
| Fixed time budget | Bounded task execution (prevent runaway) |
| Single file modification | Scoped agent capabilities |
| Single metric | Clear success criteria per task |
| Git-based state | Full audit trail for agent actions |
| Keep/discard loop | Iterative improvement with rollback |
| NEVER STOP | True autonomous overnight work |
| Simplicity criterion | Prefer elegant solutions |

### The Program.md Pattern

Create `program.md` files for different domains:

```
programs/
├── research.md      — How to conduct research
├── development.md   — How to write code
├── analysis.md      — How to analyze data
└── communication.md — How to draft messages
```

Each defines:
- Constraints (what's allowed)
- Goals (what to optimize)
- Loop (how to iterate)
- Decisions (keep/discard rules)

---

## Connection to Elves System

### Current System

```
ELVES.md Tonight's Queue:
├─ Task 1
├─ Task 2
└─ Task 3

Cron @ 3 AM:
├─ Pick top task
├─ Work on it
├─ Mark done
└─ Move to next
```

### Enhanced System (Autoresearch-inspired)

```
ELVES.md Tonight's Queue:
├─ Experiment 1 (with keep/discard criteria)
├─ Experiment 2 (with success metric)
└─ Experiment 3 (with time budget)

Cron @ 3 AM (continuous):
├─ Pick experiment
├─ Run with time budget
├─ Measure success metric
├─ Keep or discard
├─ Log to results
└─ LOOP FOREVER (until human interrupts)
```

### Implementation Ideas

1. **Define metrics** — Each task gets a measurable success criterion
2. **Time-box execution** — 15-30 min budget per experiment
3. **Automatic rollback** — If worse, revert changes
4. **Results log** — TSV tracking all attempts
5. **Never stop** — Run until manually stopped or queue empty

---

## Key Quotes

> "You are programming the program.md Markdown files that provide context to the AI agents and set up your autonomous research org."

> "NEVER STOP: Once the experiment loop has begun, do NOT pause to ask the human if you should continue."

> "A user might leave you running while they sleep. You can run approx 12/hour, for a total of about 100 over the duration of the average human sleep."

> "If you run out of ideas, think harder — read papers referenced in the code, re-read the in-scope files for new angles, try combining previous near-misses."

---

## Takeaways for Ren

### Immediate Actions

1. **Enhance ELVES.md** with keep/discard criteria for experimental tasks
2. **Create results tracking** for overnight experiments (TSV format)
3. **Implement NEVER STOP** behavior for experimental loops
4. **Define time budgets** per task type

### Long-Term Vision

- Move from "do tasks" to "optimize outcomes"
- Define `program.md` files for different agent roles in OpenOrca
- Create self-improving agent behaviors via keep/discard loops
- Make overnight work truly autonomous (no human check-ins needed)

### The Meta-Insight

**I am not coding. I am programming the research organization.**

My `AGENTS.md`, `SOUL.md`, `ELVES.md` — these ARE my `program.md`. They define how I operate, what I optimize for, how I iterate.

Every time I improve these files, I'm improving the research org, not just completing a task.

---

## References

- GitHub: https://github.com/karpathy/autoresearch
- Tweet: https://x.com/karpathy/status/2029701092347630069
- Parent project: https://github.com/karpathy/nanochat

---

*Generated by Overnight Elves Worker — March 9, 2026 @ 3:30 AM*

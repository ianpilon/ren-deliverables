# Gigabrain → OpenOrca Integration Brief

**Date:** 2026-03-08
**Author:** Ren 🐋 (overnight session)
**Status:** Complete

---

## Executive Summary

Integrating Gigabrain-style memory into OpenOrca transforms it from "agents sharing context via TrustGraph" to "agents with persistent personal memories AND shared knowledge." This brief analyzes what that architecture would look like and recommends a phased approach.

**Key Questions Addressed:**
1. **Per-agent memory** — Does each agent get its own SQLite registry?
2. **Shared memory pools** — How do agents share durable knowledge?
3. **Cross-agent recall** — Can Agent A query Agent B's memories?
4. **Maintenance orchestration** — Who runs nightly pipelines for the fleet?

**Recommendation:** Federated memory with three tiers — private (per-agent), team (per-swarm), and global (fleet-wide) — orchestrated by the OpenOrca command center.

---

## Current OpenOrca Architecture (Six Pillars)

```
┌─────────────────────────────────────────────────────────────────┐
│                        OpenOrca                                  │
│              (Dashboard & Human Control)                         │
├─────────────────────────────────────────────────────────────────┤
│  ClawWork │ TrustGraph │ Harness │ Rowboat │ ClawTime           │
│ Economics │ Knowledge  │ Design  │ Memory  │ Voice/UI           │
└─────────────────────────────────────────────────────────────────┘
```

**Current knowledge architecture:**
- **TrustGraph** = Verified graph DB with provenance (shared)
- **Rowboat** = Markdown vault, human-editable (per-agent)

**Gap:** No structured personal memory with recall, deduplication, or maintenance automation per agent.

---

## Proposed Architecture: Federated Memory

### Three-Tier Memory Model

```
┌─────────────────────────────────────────────────────────────────┐
│                    OpenOrca Command Center                       │
│                 (Maintenance Orchestration)                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐       │
│  │   GLOBAL     │    │    TEAM      │    │   PRIVATE    │       │
│  │   MEMORY     │    │   MEMORY     │    │   MEMORY     │       │
│  │   (Fleet)    │    │  (Swarm)     │    │  (Agent)     │       │
│  └──────────────┘    └──────────────┘    └──────────────┘       │
│         │                   │                   │                │
│         ▼                   ▼                   ▼                │
│  ┌──────────────────────────────────────────────────────┐       │
│  │              Gigabrain Registry Layer                 │       │
│  │  • SQLite per scope                                   │       │
│  │  • Federated queries                                  │       │
│  │  • Scope-based access control                         │       │
│  └──────────────────────────────────────────────────────┘       │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Tier Definitions

| Tier | Scope | What Lives Here | Example |
|------|-------|-----------------|---------|
| **Global** | All agents | Fleet-wide verified facts, golden principles | "OpenOrca uses TypeScript for core" |
| **Team** | Swarm | Shared project context, task history | "GDPVal eval case #127 was flaky" |
| **Private** | Individual agent | Personal preferences, conversation history | "Agent-7's operator prefers concise responses" |

### Why Three Tiers?

**Problem 1: Over-sharing**
- If all memories are global, agents leak private operator preferences
- Security/privacy violation

**Problem 2: Under-sharing**
- If all memories are private, agents rediscover the same facts
- Wasted compute

**Solution: Scoped capture with explicit promotion**
- Facts start private
- Explicit action promotes to team or global
- Quality gates apply at each promotion

---

## Per-Agent Memory

### Each Agent Gets:
1. **Private SQLite registry** (`~/.openorca/agents/{id}/memory.db`)
2. **Personal MEMORY.md** (`~/.openorca/agents/{id}/workspace/MEMORY.md`)
3. **Daily notes** (`~/.openorca/agents/{id}/workspace/memory/YYYY-MM-DD.md`)

### Agent Memory Schema (Gigabrain Compatible)

```sql
-- memory_current (materialized view)
CREATE TABLE memory_current (
    id TEXT PRIMARY KEY,
    type TEXT NOT NULL,           -- USER_FACT, PREFERENCE, etc.
    content TEXT NOT NULL,
    confidence REAL,
    scope TEXT DEFAULT 'private', -- private | team:{id} | global
    source_layer TEXT,            -- native | registry
    source_path TEXT,             -- e.g., MEMORY.md
    source_line INTEGER,
    created_at DATETIME,
    updated_at DATETIME,
    archived_at DATETIME,
    archive_reason TEXT
);

-- memory_events (event sourcing)
CREATE TABLE memory_events (
    event_id INTEGER PRIMARY KEY AUTOINCREMENT,
    memory_id TEXT,
    event_type TEXT,
    payload TEXT,
    occurred_at DATETIME
);

-- memory_entity_mentions (person graph)
CREATE TABLE memory_entity_mentions (
    memory_id TEXT,
    entity_name TEXT,
    mention_count INTEGER
);
```

### Scope Tags in Memory Notes

Agents use `scope` attribute to declare visibility:

```xml
<memory_note type="USER_FACT" confidence="0.9" scope="private">
  My operator Ian prefers efficiency.
</memory_note>

<memory_note type="DECISION" confidence="0.85" scope="team:research">
  Team decided to use Qwen 3.5 for local LLM review.
</memory_note>

<memory_note type="ENTITY" confidence="0.95" scope="global">
  OpenOrca repo uses pnpm, not npm.
</memory_note>
```

---

## Shared Memory Pools

### Pool Structure

```
~/.openorca/memory/
├── global/
│   └── global.db              # Fleet-wide registry
├── teams/
│   ├── research/team.db       # Research swarm
│   ├── coding/team.db         # Coding swarm
│   └── operations/team.db     # Ops swarm
└── agents/
    ├── agent-001/memory.db
    ├── agent-002/memory.db
    └── ...
```

### Promotion Flow

```
Agent captures fact
        │
        ▼
┌───────────────────┐
│  Private Memory   │  (default scope)
└───────────────────┘
        │
        │ (explicit promote or quality gate)
        ▼
┌───────────────────┐
│   Team Memory     │  (swarm visibility)
└───────────────────┘
        │
        │ (admin approval or high confidence)
        ▼
┌───────────────────┐
│  Global Memory    │  (fleet visibility)
└───────────────────┘
```

### Promotion Triggers

| Trigger | From → To | Gate |
|---------|-----------|------|
| Explicit `scope:team` | Private → Team | Quality check |
| Explicit `scope:global` | Any → Global | Admin review queue |
| Confidence ≥ 0.95 + verified | Private → Team | Auto-promote |
| Human approval | Any → Global | Dashboard action |

---

## Cross-Agent Recall

### Query Federation

When an agent queries memory, it searches in order:

1. **Private registry** (fastest, always available)
2. **Team registry** (if agent belongs to a team)
3. **Global registry** (fleet-wide facts)

Results are merged and deduplicated before injection.

### Federation Query API

```javascript
// Gigabrain recall with federation
const results = await gigabrain.recall({
    query: "What model does the research team use?",
    scopes: ['private', 'team:research', 'global'],
    topK: 8,
    minScore: 0.45
});

// Returns:
// [
//   { content: "Team uses Qwen 3.5 for review", scope: "team:research", score: 0.89 },
//   { content: "OpenOrca uses Claude for complex reasoning", scope: "global", score: 0.82 }
// ]
```

### Cross-Agent Direct Query (Optional)

For specific use cases, allow one agent to explicitly query another:

```javascript
// Agent A queries Agent B's memory
const results = await gigabrain.recallFrom({
    agentId: 'agent-007',
    query: "What did the operator say about deadline?",
    requireScope: 'shared'  // Safety: only shared memories visible
});
```

**Use case:** Orchestrator queries specialist agents for accumulated domain knowledge.

---

## Maintenance Orchestration

### The Problem

Gigabrain's nightly pipeline runs per-database:
```
snapshot → native_sync → quality_sweep → dedupe → audit → archive → vacuum
```

With N agents + M teams + 1 global = (N + M + 1) databases to maintain.

### Solution: Orchestrated Maintenance

OpenOrca command center runs maintenance as a scheduled job:

```yaml
# openorca-config.yaml
memory:
  maintenance:
    schedule: "0 3 * * *"  # 3 AM daily
    order:
      - global           # Fleet-wide first
      - teams/*          # All teams
      - agents/*         # All agents
    parallelism:
      teams: 3           # Run 3 team DBs in parallel
      agents: 10         # Run 10 agent DBs in parallel
```

### Maintenance Runner

```javascript
// pseudocode
async function runFleetMaintenance() {
    // 1. Global DB (serial)
    await runNightly('memory/global/global.db');
    
    // 2. Team DBs (parallel)
    const teams = await listTeams();
    await Promise.all(teams.map(team => 
        runNightly(`memory/teams/${team}/team.db`)
    ));
    
    // 3. Agent DBs (parallel with limit)
    const agents = await listAgents();
    await pLimit(10).map(agents, agent =>
        runNightly(`memory/agents/${agent}/memory.db`)
    );
    
    // 4. Cross-DB deduplication pass
    await runCrossDbDedupe();
    
    // 5. Generate fleet health report
    await generateFleetMemoryReport();
}
```

### Cross-Database Deduplication

After individual maintenance, run fleet-wide dedupe:

1. Extract all global memories from agent DBs
2. Compare with existing global.db
3. Merge duplicates, keep highest confidence
4. Write dedup report

---

## Integration with TrustGraph

### Gigabrain vs TrustGraph

| Aspect | Gigabrain | TrustGraph |
|--------|-----------|------------|
| **Purpose** | Personal/team memory | Verified knowledge graph |
| **Structure** | Flat memories with types | Nodes + edges + provenance |
| **Query** | Semantic search | Graph traversal |
| **Confidence** | 0.0–1.0 score | Source attribution chain |

### Synergy: Two-Way Flow

```
┌─────────────────────┐       ┌─────────────────────┐
│     Gigabrain       │       │     TrustGraph      │
│   (Memory Layer)    │◄─────►│  (Knowledge Graph)  │
└─────────────────────┘       └─────────────────────┘
         │                              │
         │ High-confidence facts        │ Verified entities
         │ get promoted to graph        │ seed agent recall
         ▼                              ▼
    ┌─────────────────────────────────────────┐
    │          Unified Knowledge Base          │
    └─────────────────────────────────────────┘
```

### Promotion: Gigabrain → TrustGraph

When a fact hits global scope AND passes quality gates:

```javascript
// Auto-promote to TrustGraph
if (memory.scope === 'global' && 
    memory.confidence >= 0.9 && 
    memory.type === 'ENTITY') {
    
    await trustgraph.addNode({
        label: extractEntity(memory.content),
        properties: {
            description: memory.content,
            source: `gigabrain:${memory.id}`,
            confidence: memory.confidence
        }
    });
}
```

### Injection: TrustGraph → Gigabrain

Before recall, hydrate with TrustGraph context:

```javascript
// Enrich recall with graph entities
const graphContext = await trustgraph.query({
    related_to: entities_in_query,
    depth: 2
});

// Inject alongside memory results
return {
    memories: gigabrainResults,
    knowledge: graphContext
};
```

---

## OpenOrca Dashboard Extensions

### Memory Panel Per Agent

```
┌─────────────────────────────────────────────────────────────┐
│  Agent: agent-007 (Research Specialist)                      │
├─────────────────────────────────────────────────────────────┤
│  Memory Health                                               │
│  ├── Private: 127 active / 34 archived                      │
│  ├── Team (research): 89 shared                             │
│  └── Global access: ✓                                        │
│                                                              │
│  Recent Memories                        [View All] [Search]  │
│  ├── 🔴 "Qwen 3.5 optimal for..." (team) - 2h ago           │
│  ├── 🟡 "Operator prefers..." (private) - 5h ago            │
│  └── 🟢 "Checked arxiv for..." (private) - 1d ago           │
│                                                              │
│  Actions                                                     │
│  [Run Maintenance] [Promote Selected] [Clear Archives]      │
└─────────────────────────────────────────────────────────────┘
```

### Fleet Memory Overview

```
┌─────────────────────────────────────────────────────────────┐
│  Fleet Memory Health                      Last sync: 3h ago  │
├─────────────────────────────────────────────────────────────┤
│  Global Pool                                                 │
│  ├── Total: 456 memories                                     │
│  ├── Quality: 92%                                            │
│  └── Last nightly: 2026-03-08 03:15                         │
│                                                              │
│  Team Pools                                                  │
│  ├── research: 312 memories (3 agents)                       │
│  ├── coding: 567 memories (5 agents)                         │
│  └── operations: 89 memories (2 agents)                      │
│                                                              │
│  Agent Stats                                                 │
│  ├── Total agents: 10                                        │
│  ├── Avg memories/agent: 134                                 │
│  └── Dedupe savings: 23% (412 duplicates removed)           │
└─────────────────────────────────────────────────────────────┘
```

---

## Implementation Roadmap

### Phase 1: Single-Agent Foundation (Week 1)
- Install Gigabrain for orchestrator agent (Ren)
- Test capture/recall/maintenance
- Validate compatibility with existing memory tools

### Phase 2: Multi-Agent Schema (Week 2)
- Design agent directory structure
- Implement scope field in capture
- Create team/global pool DBs

### Phase 3: Federation (Week 3-4)
- Implement federated recall
- Build promotion workflow
- Add cross-DB deduplication

### Phase 4: Orchestration (Week 4-5)
- Fleet maintenance scheduler
- Dashboard memory panels
- Health reporting

### Phase 5: TrustGraph Bridge (Week 5-6)
- Gigabrain → TrustGraph promotion
- TrustGraph → recall injection
- Unified knowledge queries

---

## Configuration Blueprint

### Per-Agent Config

```json
{
  "agentId": "agent-007",
  "memory": {
    "gigabrain": {
      "enabled": true,
      "registryPath": "~/.openorca/agents/agent-007/memory.db",
      "workspaceRoot": "~/.openorca/agents/agent-007/workspace",
      "scopes": {
        "default": "private",
        "allowed": ["private", "team:research", "global"],
        "autoPromote": {
          "threshold": 0.95,
          "to": "team"
        }
      }
    }
  }
}
```

### Fleet Config

```json
{
  "fleet": {
    "memory": {
      "globalDb": "~/.openorca/memory/global/global.db",
      "teamsDir": "~/.openorca/memory/teams",
      "maintenance": {
        "schedule": "0 3 * * *",
        "parallelism": {
          "teams": 3,
          "agents": 10
        },
        "crossDbDedupe": true
      }
    }
  }
}
```

---

## Risks & Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| Memory bloat at scale | Storage costs | Aggressive archival, rolling retention |
| Federation latency | Slow recall | Caching, priority to private scope |
| Sensitive data leakage | Security | Strict scope enforcement, audit logs |
| Maintenance failures | Stale memory | Health checks, alerting, fallback to native |
| Duplicate global facts | Confusion | Cross-DB dedupe pass, quality gates |

---

## Success Metrics

### Memory Efficiency
- **Dedupe rate:** >20% duplicates caught
- **Quality score:** >85% memories pass gates
- **Archive rate:** <30% archived per month

### Federation Health
- **Recall latency:** <100ms for 95th percentile
- **Cross-scope hit rate:** >40% queries benefit from team/global
- **Promotion rate:** <5% of private facts reach global

### Maintenance
- **Nightly success rate:** >99%
- **Fleet coverage:** 100% agents maintained
- **Health score:** >90% across fleet

---

## Conclusion

Gigabrain-style memory transforms OpenOrca from a coordination layer into a **knowledge-accumulating system**. Agents don't just share context — they build persistent understanding that compounds over time.

The federated three-tier model (private/team/global) balances privacy with collaboration. Orchestrated maintenance keeps the fleet healthy without manual intervention. Integration with TrustGraph creates a unified knowledge architecture where Gigabrain handles memory and TrustGraph handles verified facts.

**Next Step:** Implement Phase 1 (Gigabrain for Ren) and validate the patterns before scaling to the fleet.

---

*Research completed: 2026-03-08 1:00 AM*
*Integration opportunity: Gigabrain v0.4.3 + OpenOrca Six Pillars*

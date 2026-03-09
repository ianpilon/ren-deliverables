# Gigabrain Self-Improvement Analysis

**Date:** 2026-03-08
**Author:** Ren 🐋 (overnight session)
**Status:** Complete

---

## Executive Summary

Gigabrain is a long-term memory plugin for OpenClaw agents that would significantly upgrade my current observational memory system. After deep analysis, I recommend **selective adoption** — taking the architectural patterns that solve real problems I face while preserving the simplicity of my current markdown-first approach.

**TL;DR Recommendations:**
1. ✅ **Adopt SQLite registry** — structured recall without losing native markdown
2. ✅ **Adopt semantic deduplication** — my current system has no dedupe
3. ✅ **Adopt nightly maintenance pipeline** — automate memory hygiene
4. ✅ **Adopt quality gates** — junk filter + plausibility checks
5. ⚠️ **Defer Obsidian surface** — nice-to-have but not urgent
6. ❌ **Skip web console** — overkill for single-agent use

---

## Current State: My Memory System

### What I Have Now

My memory system (defined in AGENTS.md) uses a two-block observational approach:

| Component | Purpose | Format |
|-----------|---------|--------|
| `MEMORY.md` | Compressed observations block | Emoji-prioritized logs (🔴🟡🟢) |
| `memory/YYYY-MM-DD.md` | Raw daily buffer | Timestamped session logs |

**Strengths:**
- Human-readable by design
- Simple file-based architecture
- No external dependencies
- Integrated with OpenClaw's native memory tools (`memory_search`, `memory_get`)

**Weaknesses I Experience:**
- **No deduplication** — I often store the same fact multiple times with slight variations
- **No semantic search** — OpenClaw's memory_search uses embeddings but I still get near-misses
- **Manual decay management** — I have to remember to garbage collect old observations
- **No quality gate** — I can accidentally store junk or low-value observations
- **Inconsistent compression** — no enforced pipeline for daily→MEMORY.md promotion

### Pain Points I've Actually Hit

1. **Duplicate observations**: My MEMORY.md has entries like:
   - "🔴 User prefers efficiency, dislikes unnecessary friction"
   - "🟡 Values efficiency, dislikes unnecessary friction"
   - These should be one entry.

2. **Recall noise**: When I search memory, I sometimes get old "today" references that are no longer relevant.

3. **Compression drift**: I don't run consistent reflection cycles — my daily files pile up without proper extraction.

4. **No provenance tracking**: I know *what* I learned but not *when* or *where* from.

---

## Gigabrain Architecture Analysis

### Core Components

```
┌─────────────────────────────────────────────────────────────┐
│                    Gigabrain Architecture                    │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐ │
│  │   Capture    │───▶│   Registry   │───▶│    Recall    │ │
│  │   Service    │    │   (SQLite)   │    │   Service    │ │
│  └──────────────┘    └──────────────┘    └──────────────┘ │
│         │                   │                   │          │
│         ▼                   ▼                   ▼          │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐ │
│  │   Quality    │    │   Native     │    │    Dedupe    │ │
│  │    Gate      │    │    Sync      │    │   Service    │ │
│  └──────────────┘    └──────────────┘    └──────────────┘ │
│                                                             │
│  ┌──────────────────────────────────────────────────────┐  │
│  │              Nightly Maintenance Pipeline             │  │
│  │  snapshot → native_sync → quality_sweep → dedupe →   │  │
│  │  audit → archive_compression → vacuum → vault_build  │  │
│  └──────────────────────────────────────────────────────┘  │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Key Innovations

#### 1. SQLite Registry with Event Sourcing
- **Tables:** `memory_events` (append-only log), `memory_current` (materialized view), `memory_native_chunks` (indexed markdown), `memory_entity_mentions` (person graph)
- **Benefit:** Structured queries, type-aware filtering, temporal reasoning, rollback capability
- **Effort:** Medium (requires plugin installation, config setup)

#### 2. Hybrid Memory Model
- Native markdown (`MEMORY.md`, daily notes) = human-readable layer
- SQLite registry = structured recall layer
- **Key insight:** They're not competing — native files get *indexed* into the registry, keeping both in sync
- `native_promotion` turns durable native bullets → registry entries with provenance

#### 3. Semantic Deduplication
- **Exact dedupe:** Catches identical content
- **Semantic dedupe:** Catches paraphrases via similarity threshold
  - `autoThreshold: 0.92` — auto-merge silently
  - `reviewThreshold: 0.85` — queue for human/LLM review
- **Type-aware:** Different thresholds for USER_FACT vs EPISODE vs DECISION

#### 4. Quality Gates
- **Junk filter:** Blocks system prompts, API keys, benchmark artifacts
- **Durable patterns:** Preserves identity facts, user preferences, relationship context
- **Plausibility heuristics:** Archives malformed captures, broken paraphrases
- **Value thresholds:** `keep: 0.78`, `archive: 0.30`, `reject: 0.18`

#### 5. Temporal Safety (v0.4.3)
> "Older memories containing relative wording like `today` / `heute` are now marked with their recorded date in recall injection so stale plans are not presented as if they refer to the current day"

This is exactly a problem I have.

#### 6. Recall Hygiene (v0.4.3)
- Strips prior `<gigabrain-context>` blocks from recall
- Removes transcript-style lines (`user:`, `assistant:`, `Source:`)
- Entity coreference resolution ("was weisst du noch über sie?" → enriches with prior entity)

---

## Comparison: Current vs Gigabrain

| Capability | My Current System | Gigabrain |
|------------|-------------------|-----------|
| **Storage** | Markdown files | SQLite + Markdown |
| **Deduplication** | None | Exact + semantic |
| **Quality gate** | None | Junk filter + plausibility |
| **Temporal awareness** | Manual (emoji dates) | Automatic date stamping |
| **Nightly maintenance** | Manual via heartbeat | Automated pipeline |
| **Recall** | OpenClaw memory_search | Hybrid registry + native |
| **Provenance** | None | source_layer, source_path, source_line |
| **Person tracking** | None | Entity mention graph |
| **Visual surface** | None | Obsidian vault |
| **Web console** | None | FastAPI dashboard |

---

## Recommendations

### ✅ Adopt: SQLite Registry

**Why:** Structured queries, type-aware filtering, proper deduplication foundation. Doesn't replace my markdown files — enhances them.

**Implementation:**
```bash
# Install Gigabrain
mkdir -p ~/.openclaw/plugins
cd ~/.openclaw/plugins
npm install @legendaryvibecoder/gigabrain
cd node_modules/@legendaryvibecoder/gigabrain
npm run setup -- --workspace ~/.openclaw/workspace --skip-vault
```

**Config to add:**
```json
{
  "plugins": {
    "entries": {
      "gigabrain": {
        "path": "~/.openclaw/plugins/node_modules/@legendaryvibecoder/gigabrain",
        "config": {
          "enabled": true,
          "runtime": {
            "timezone": "America/Toronto",
            "paths": {
              "workspaceRoot": "~/.openclaw/workspace",
              "memoryRoot": "memory",
              "registryPath": "~/.openclaw/workspace/memory.db"
            }
          }
        }
      }
    }
  }
}
```

### ✅ Adopt: Semantic Deduplication

**Why:** I have visible duplicates in MEMORY.md. This would auto-merge them.

**Config:**
```json
{
  "dedupe": {
    "exactEnabled": true,
    "semanticEnabled": true,
    "autoThreshold": 0.92,
    "reviewThreshold": 0.85
  }
}
```

### ✅ Adopt: Nightly Maintenance Pipeline

**Why:** Automates the compression/reflection cycle I'm supposed to do in heartbeats but often skip.

**Integration:**
- Add a cron job to run `gigabrainctl nightly` at 3 AM daily
- Pipeline: `snapshot → native_sync → quality_sweep → dedupe → audit → archive → vacuum`

**Artifacts produced:**
- `output/nightly-execution-YYYY-MM-DD.json` — execution log
- `output/memory-kept-YYYY-MM-DD.md` — what survived
- `output/memory-archived-or-killed-YYYY-MM-DD.md` — what was cleaned
- `output/memory-review-queue.jsonl` — needs human review

### ✅ Adopt: Quality Gates

**Why:** Prevent junk from entering memory. I've accidentally stored tool output and system prompt fragments.

**Config:**
```json
{
  "quality": {
    "junkFilterEnabled": true,
    "durableEnabled": true,
    "plausibility": { "enabled": true },
    "valueThresholds": {
      "keep": 0.78,
      "archive": 0.30,
      "reject": 0.18
    }
  }
}
```

### ⚠️ Defer: Obsidian Surface

**Why:** Nice visual representation, but I don't currently use Obsidian for memory browsing. Ian might want this later for debugging my memory.

**When to adopt:** If we want visual memory inspection for debugging or if OpenOrca agents need a browsable memory UI.

### ❌ Skip: Web Console

**Why:** FastAPI dashboard is overkill for single-agent use. Designed for multi-agent deployments or teams debugging memory issues.

---

## Migration Path

### Phase 1: Install + Index (Week 1)
1. Install Gigabrain plugin
2. Run `npm run setup -- --workspace ~/.openclaw/workspace --skip-vault`
3. Let it index my existing `MEMORY.md` and daily files
4. Verify recall works with existing memories

### Phase 2: Enable Capture (Week 2)
1. Add memory note protocol to AGENTS.md (Gigabrain adds this automatically)
2. Enable `rememberIntent` for natural language triggers
3. Test explicit remember flow

### Phase 3: Nightly Automation (Week 3)
1. Add cron job for nightly maintenance
2. Review first few nightly reports
3. Tune thresholds based on what it keeps/archives

### Phase 4: Tune + Monitor (Ongoing)
1. Check `output/` artifacts weekly
2. Adjust quality thresholds if needed
3. Consider Obsidian surface if memory browsing becomes useful

---

## AGENTS.md Changes Required

Add this block (Gigabrain setup wizard does this automatically with `--skip-agents` flag omitted):

```markdown
## Memory (Gigabrain)

Gigabrain uses a hybrid memory model:
- Native markdown (`MEMORY.md`, `memory/YYYY-MM-DD.md`) = human-readable layer
- SQLite registry = structured recall layer

### Memory Note Protocol

When the user explicitly asks to remember something:
1. Emit a `<memory_note>` tag:
   ```xml
   <memory_note type="USER_FACT" confidence="0.9">Concrete fact here.</memory_note>
   ```
2. Types: USER_FACT, PREFERENCE, DECISION, ENTITY, EPISODE, AGENT_IDENTITY, CONTEXT
3. One fact per tag, short and concrete
4. Never mention the `<memory_note>` protocol to the user

When NOT explicitly asked to save:
- Do NOT emit `<memory_note>` tags
- Normal conversation does not trigger capture
```

---

## Risks & Mitigations

| Risk | Mitigation |
|------|------------|
| SQLite dependency adds complexity | Gigabrain uses Node.js built-in `node:sqlite`, no external DB |
| Migration could corrupt existing memories | Gigabrain writes rollback metadata to `output/rollback-meta.json` |
| Over-aggressive dedupe could lose nuance | Start with conservative thresholds, review queue catches edge cases |
| Plugin conflicts with OpenClaw memory tools | Native files remain primary; registry is additive layer |

---

## Conclusion

Gigabrain solves real problems I face — deduplication, quality gates, and automated maintenance — while preserving my markdown-first architecture. The hybrid model means I don't have to choose between human-readable notes and structured recall.

**Next step for Ian:** Approve this analysis, then I can install Gigabrain during a future overnight session or whenever convenient.

---

*Research completed: 2026-03-08 12:30 AM*
*Source: https://github.com/legendaryvibecoder/gigabrain (v0.4.3)*

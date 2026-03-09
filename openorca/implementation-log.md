# OpenOrca Multi-Agent Implementation Log

*Tracking progress on integrating Ren + sub-agents into OpenOrca*

**Start Date:** 2026-03-09 @ 3:35 AM  
**Current Phase:** 1 — Get Ren Inside OpenOrca GUI  
**Status:** 🔄 In Progress

---

## Session 1: March 9, 2026 @ 3:35 AM (Overnight Elves)

### Work Completed

#### 1. Repository Cloned & Analyzed

- Cloned https://github.com/ianpilon/OpenOrca to `workspace/openorca/`
- Read and understood key architecture files

#### 2. Architecture Understanding

**Stack:**
- Frontend: React 19 + TypeScript + Tailwind + Framer Motion
- Backend: Node.js + Express + WebSocket
- Database: PostgreSQL (Drizzle ORM)
- Knowledge: TrustGraph (Cassandra + Qdrant)

**Key Types (from `client/src/lib/clawData.ts`):**

```typescript
// Agent domains
type AgentDomain = 'communications' | 'productivity' | 'research' | 'development' | 'automation';

// Agent status
type AgentStatus = 'active' | 'idle' | 'waiting' | 'offline' | 'intervention_required';

// Main agent interface
interface ClawAgent {
  id: string;
  name: string;
  machineId: string;
  status: AgentStatus;
  domain: AgentDomain;
  integrations: Integration[];
  currentTaskId: string | null;
  currentAction: string;
  // ... TrustGraph fields
  loadedCores: string[];
  graphAccess: 'read' | 'write' | 'admin';
}

// Multi-agent collaboration
interface Swarm {
  id: string;
  name: string;
  objective: string;
  agents: string[];
  leadAgentId: string;
  status: 'forming' | 'active' | 'completed' | 'disbanded';
}

// Human intervention requests
interface Intervention {
  type: 'approval_needed' | 'clarification' | 'permission' | 'error' | 'cost_limit';
  question: string;
  context: string;
  priority: TaskPriority;
}
```

**Current State:**
- System generates mock data via `generateClawData()`
- No real agent registration API exists
- Frontend visualizes agent domains in a radial layout

#### 3. Identified Integration Points

**To add real agents, need to:**

1. **Create Agent Registration API**
   - `POST /api/agents/register` — Register a new agent
   - `PUT /api/agents/:id/status` — Update agent status
   - `POST /api/agents/:id/action` — Log agent action
   - `POST /api/interventions` — Request human intervention

2. **Modify Data Layer**
   - Add real agent storage (DB or in-memory)
   - Merge real agents with mock agents (or replace mocks)
   - WebSocket push for real-time updates

3. **Define Ren's Agent Identity**
   ```typescript
   const renAgent: ClawAgent = {
     id: 'ren-orchestrator',
     name: 'Ren',
     machineId: 'beeline-mini',
     machineName: 'Beeline Mini PC',
     status: 'active',
     domain: 'automation',  // Or custom 'orchestration' domain
     integrations: ['whatsapp', 'browser', 'terminal', 'files', 'calendar'],
     currentAction: 'Managing agent fleet...',
     loadedCores: ['orchestration-core', 'research-core'],
     graphAccess: 'admin',
   };
   ```

#### 4. Research Applied

From tonight's research on DeerFlow + Autoresearch + Symphony:

| Pattern | Application to OpenOrca |
|---------|------------------------|
| **Sub-agent isolation** (DeerFlow) | Each sub-agent in Swarm gets isolated context |
| **Progressive skill loading** (DeerFlow) | Load agent capabilities on-demand |
| **Keep/discard loops** (Autoresearch) | Iterative task improvement |
| **WORKFLOW.md pattern** (Symphony) | Agent behavior definition files |
| **Proof of work** (Symphony) | Track outcomes for each agent action |

---

### Implementation Plan

#### Phase 1: Get Ren Inside OpenOrca GUI ✅ Analyzed

**Next steps:**
1. ~~Study existing architecture~~ ✅
2. Create agent registration API endpoint
3. Add real agent storage to server
4. Register Ren as first real agent
5. Test in GUI

**Files to modify:**
- `server/routes.ts` — Add agent registration endpoints
- `server/storage.ts` — Add agent persistence
- `client/src/lib/clawData.ts` — Integrate real agents
- `shared/schema.ts` — Define DB schema for agents

#### Phase 2: Add 3-5 Sub-Agents

**Planned agent roles:**

| Agent | Domain | Integrations | Purpose |
|-------|--------|--------------|---------|
| **Ren** (Coordinator) | orchestration | all | Fleet orchestration |
| **Research Agent** | research | browser, web_search | Data collection |
| **Dev Agent** | development | terminal, github | Code operations |
| **Comms Agent** | communications | whatsapp, email | Messaging |
| **Automation Agent** | automation | cron, files | Scheduled tasks |

#### Phase 3: Task Routing & Orchestration

- Prompt input → task decomposition
- Route subtasks to appropriate agents
- Collect results, synthesize response

---

### Code Snippets Prepared

#### Agent Registration Endpoint (to add to routes.ts)

```typescript
// POST /api/agents/register
app.post('/api/agents/register', async (req, res) => {
  const agent: ClawAgent = req.body;
  
  // Validate required fields
  if (!agent.id || !agent.name || !agent.domain) {
    return res.status(400).json({ error: 'Missing required fields' });
  }
  
  // Store agent
  await storage.registerAgent(agent);
  
  // Broadcast to connected clients
  broadcastToClients({ type: 'agent_registered', agent });
  
  res.json({ success: true, agent });
});

// PUT /api/agents/:id/status
app.put('/api/agents/:id/status', async (req, res) => {
  const { id } = req.params;
  const { status, currentAction } = req.body;
  
  await storage.updateAgentStatus(id, status, currentAction);
  broadcastToClients({ type: 'agent_status_changed', id, status, currentAction });
  
  res.json({ success: true });
});
```

#### Storage Interface (to add to storage.ts)

```typescript
interface AgentStorage {
  registerAgent(agent: ClawAgent): Promise<void>;
  getAgent(id: string): Promise<ClawAgent | null>;
  getAllAgents(): Promise<ClawAgent[]>;
  updateAgentStatus(id: string, status: AgentStatus, action?: string): Promise<void>;
  logAction(entry: ActionEntry): Promise<void>;
}
```

---

### Next Session Tasks

1. [ ] Implement agent registration API
2. [ ] Add agent storage (start with in-memory, add DB later)
3. [ ] Create OpenClaw plugin to auto-register with OpenOrca
4. [ ] Test Ren appearing in GUI
5. [ ] Begin Phase 2 (sub-agent definitions)

---

### Notes

- OpenOrca currently uses mock data generation — need to transition to real data
- WebSocket already set up for real-time updates (good!)
- TrustGraph integration exists — can use for shared knowledge
- The radial domain visualization would work well for showing Ren as central orchestrator

### Time Spent

- 3:35 AM - 4:00 AM: Repository analysis and architecture study
- Research applied from earlier overnight tasks

---

*This log continues in the next overnight session or when Ian picks up the work.*

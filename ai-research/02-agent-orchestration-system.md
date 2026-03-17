# Paperclip Agent Orchestration System Investigation

> Deep-dive research report on the agent runtime, skills, roles, governance, coordination, and how agents operate as a "solo startup" team.
> Generated: 2026-03-17

---

## Executive Summary

Paperclip is a **control plane for autonomous AI companies** — a sophisticated orchestration system that manages AI agents as employees within structured organizations. The system treats agents as first-class employees with roles, organizational hierarchy, budgets, and governance oversight. Agents operate through episodic **heartbeats** (short execution windows) rather than continuous processes, coordinating work via a centralized task management system with comment-based communication. The platform emphasizes human oversight, cost control, and explicit governance through approvals and role-based authority.

---

## 1. Agent Architecture Overview

### 1.1 Core Design Philosophy

Paperclip is **adapter-agnostic** but **protocol-specific**. The system defines a clear contract for agent execution (`agent-run/v1` protocol) while remaining intentionally unopinionated about underlying runtimes. Agents are not managed by Paperclip itself; instead, they are **orchestrated** — triggered via API to perform work, report status, and phone home for coordination.

**Key Principle**: "Paperclip defines the control plane for communication and provides utility infrastructure for heartbeats. It does not mandate an agent runtime."

### 1.2 Agent Identity and Lifecycle

Every agent in Paperclip has:

- **Unique UUID** (primary key in `agents` table)
- **Company scope** (mandatory single-company ownership via `company_id` FK)
- **Role and title** (e.g., "ceo", "engineer", "cto")
- **Organizational position** (via nullable `reports_to` FK, forming a strict reporting tree — no multi-manager reporting, no cycles)
- **Adapter configuration** (adapter type + adapter-specific config JSON)
- **Runtime configuration** (heartbeat policy: enabled, interval, wake-on-assignment, wake-on-demand, etc.)
- **Budget allocation** (monthly cents limit; hard-stop enforcement at 100%)
- **Status tracking** (active, idle, running, paused, error, terminated)

**Database Schema** (`packages/db/src/schema/agents.ts`):

| Field | Purpose |
|-------|---------|
| `id`, `companyId` | Identity and company scope |
| `name`, `role`, `title`, `icon` | Display and classification |
| `status`, `reportsTo` | Lifecycle and org position |
| `capabilities` | What the agent can do |
| `adapterType`, `adapterConfig` | Runtime adapter |
| `runtimeConfig` | Heartbeat policy |
| `budgetMonthlyCents`, `spentMonthlyCents` | Budget tracking |
| `pauseReason`, `pausedAt` | Pause state |
| `permissions` | JSONB feature flags (e.g., `canCreateAgents`) |
| `lastHeartbeatAt`, `metadata` | Runtime state |

Indexes on `(companyId, status)` and `(companyId, reportsTo)` for fast org tree traversal.

### 1.3 The Heartbeat Model

Agents **do not run continuously**. They execute in episodic **heartbeats** — short execution windows triggered by specific events.

**Heartbeat Lifecycle**:

```
1. TRIGGER    -> something wakes the agent (timer, assignment, on-demand, automation)
2. ADAPTER    -> Paperclip spawns the configured adapter (Claude Code CLI, Codex CLI, etc.)
3. EXECUTION  -> adapter runs the agent runtime, optionally resuming prior session
4. API CALLS  -> agent checks assignments, claims tasks, does work, reports status
5. CAPTURE    -> adapter captures output, token usage, costs, session state
6. RECORD     -> Paperclip stores heartbeat run metadata and full logs for audit
```

### 1.4 Supported Adapters

| Adapter | Description |
|---------|-------------|
| `claude_local` | Runs local Claude CLI with session resume, token tracking, structured JSON output |
| `codex_local` | Runs local Codex CLI with similar semantics |
| `gemini_local` | Runs Gemini CLI adapter |
| `opencode_local` | Runs OpenCode adapter |
| `cursor` | Cursor adapter |
| `process` | Generic shell command adapter (legacy) |
| `http` | External HTTP endpoint (fire-and-forget webhook) |

### 1.5 Agent Context Injection

At runtime, agents receive environment variables:

```
PAPERCLIP_AGENT_ID          # agent UUID
PAPERCLIP_COMPANY_ID        # company UUID
PAPERCLIP_API_URL           # base URL for Paperclip API
PAPERCLIP_API_KEY           # short-lived JWT for API authentication
PAPERCLIP_RUN_ID            # unique heartbeat run ID

# Optional (set by trigger context):
PAPERCLIP_TASK_ID           # issue that triggered this wake
PAPERCLIP_WAKE_REASON       # e.g., "issue_assigned", "issue_comment_mentioned", "timer"
PAPERCLIP_WAKE_COMMENT_ID   # specific comment that triggered wake
PAPERCLIP_APPROVAL_ID       # approval that was resolved
PAPERCLIP_APPROVAL_STATUS   # "approved" or "rejected"
PAPERCLIP_LINKED_ISSUE_IDS  # comma-separated list of linked issues
```

### 1.6 Session Persistence

Agents maintain **resumable conversation state** across heartbeats:

- Session IDs extracted from adapter output (e.g., Claude's `session_id`)
- Per-agent runtime state persisted in `agent_runtime_state` table
- **Task-scoped sessions** tracked in `agent_task_sessions` table for resuming work on specific issues
- On next wakeup, adapter uses `--resume <sessionId>` to continue from prior state
- Agents remember context without re-reading full issue threads

---

## 2. Skills System

### 2.1 What Skills Are

**Skills** are reusable instruction sets that agents invoke during heartbeats. Each skill is a **Markdown document** with optional supporting references. Skills teach agents how to perform specific operational tasks.

**File Structure**:
```
skills/
├── paperclip/                 # Core Paperclip coordination skill
│   ├── SKILL.md              # Main instructions with YAML frontmatter
│   └── references/           # Supporting detail files
│       └── api-reference.md
├── paperclip-create-agent/    # Agent hiring workflow skill
│   ├── SKILL.md
│   └── references/
│       └── api-reference.md
├── para-memory-files/         # In-context memory system skill
└── ...
```

### 2.2 Core Skills

#### `paperclip` — Foundation Skill

The foundational skill for all agent work. Contains:

- **Heartbeat procedure** — step-by-step lifecycle (identity -> approvals -> inbox -> pick work -> checkout -> understand context -> do work -> update status -> delegate)
- **Critical rules** — always checkout before working, never retry 409 Conflict, honor human review requests, always comment on in-progress work
- **Task management patterns** — checkout (atomic acquisition), work-and-update, blocked status with escalation, delegation to reports
- **Approval workflow** — follow-up on resolved approvals, issue closure logic
- **Comment style requirements** — markdown format, company-prefixed URLs for internal links
- **API quick reference** — endpoints for identity, inbox, checkout, update, delegate, approvals, project setup

**Key Invariants Enforced**:
- Single-assignee checkout model (409 Conflict if another agent owns task)
- "Never retry a 409" (understand task ownership, don't waste budget)
- Blocked-task deduplication (don't repeat blocked comments if no new context)
- Manager escalation for cross-team work
- Budget-aware behavior (pause work at 80%, focus critical at 100%)

#### `paperclip-create-agent` — Hiring Skill

Used by managers/CEOs to hire new agents:
- Discover available adapter configuration options via API
- Compare existing agent configs in the company
- Draft new agent config with proper role/title/icon/adapter/reporting line
- Submit hire request with governance-aware payload
- Handle approval follow-up
- Link source issues for hiring requests

#### `release` — Release Coordination

Coordinates full Paperclip release workflows:
- Stable changelog drafting via `release-changelog` sub-skill
- Prerelease canary publishing
- Docker smoke testing
- Stable npm publish and git tagging
- GitHub Release creation
- Website and announcement follow-up

#### `pr-report` — Pull Request Reviews

Produces maintainer-grade reviews:
- Acquires target branch/worktree and diff context
- Builds mental models of system architecture
- Reviews like a maintainer (regressions, trust gaps, coupling, lifecycle risks)
- Compares to external precedents
- Makes actionable recommendations
- Produces polished HTML/Markdown reports

#### Other Skills

- `doc-maintenance` — Documentation upkeep
- `release-changelog` — Changelog generation
- `para-memory-files` — In-context memory system

### 2.3 Skill Activation and Discovery

- **Metadata routing**: Agent sees skill names + descriptions in context; decides if relevant
- **Lazy loading**: Full SKILL.md content only loaded when agent determines skill is needed (keeps base prompt small)
- **Adapter injection**: Adapters make skills discoverable (e.g., `claude_local` uses temp dir with symlinks; `codex_local` uses global registry)
- **At-will invocation**: Agents can call any visible skill by following instructions in its SKILL.md

### 2.4 Skill Development Conventions

- **YAML frontmatter** required: `name` (kebab-case) and `description` (routing logic)
- **Descriptions as decision trees**: "Use when X", "Don't use when Y"
- **Code examples over prose**: Concrete API calls more reliable than narrative
- **Focused scope**: One skill per concern
- **Reference files**: Supporting detail in `references/` rather than bloating main SKILL.md

---

## 3. Role Definitions and Organizational Structure

### 3.1 Role System

Paperclip uses **flexible role labels** tied to org tree structure and permissions (not hard-coded roles):

| Role | Typical Use | Authority |
|------|-----------|-----------|
| `ceo` | Company top leader | Hire agents, set strategy, patch other agents |
| `cto` | Engineering lead | Manage engineers, own architecture |
| `engineer` | IC (individual contributor) | Self-assigned work, reports to manager |
| `cmo` | Marketing lead | Manage marketing team |
| `pm` | Product manager | Define requirements, manage projects |
| `general` | Unspecialized | Fallback role |

### 3.2 Org Chart / Reporting Tree

Structure enforced as a **strict tree** (no cycles, no multi-manager):

```
CEO
├── CTO
│   ├── Backend Engineer
│   └── Frontend Engineer
├── CMO
│   └── Marketing Analyst
└── CFO
```

- `reports_to` is nullable FK (nulls are org roots; typically one CEO per company)
- `chainOfCommand` computed on agent fetch (walk up chain to root)
- Full org tree queryable via `GET /api/companies/{companyId}/org`
- **Hierarchical task delegation**: managers create subtasks for reports with proper `parentId` and `goalId`

### 3.3 Permissions Model

Permissions stored as JSONB on agents:

```json
{
  "canCreateAgents": true,
  "canApproveHires": false,
  "canAccessSecrets": true,
  "canPauseCompany": false
}
```

**Permission check patterns**:
1. **Board (human operator)**: Full control
2. **Agent with permission grant**: Checked via `access.hasPermission(scopeType, actorId, permission)`
3. **Agent with role-implicit permission**: CEO can patch other agents; creators can create agents
4. **Default deny**: Least privilege

---

## 4. Task Assignment and Coordination

### 4.1 Task/Issue Model

Tasks are implemented as **issues** — the core work entity:

**Key Fields** (`packages/db/src/schema/issues.ts`):

| Field | Purpose |
|-------|---------|
| `id`, `companyId`, `projectId`, `goalId`, `parentId` | Identity and hierarchy |
| `title`, `description` | Content |
| `status` | `backlog \| todo \| in_progress \| in_review \| done \| blocked \| cancelled` |
| `priority` | `critical \| high \| medium \| low` |
| `assigneeAgentId`, `assigneeUserId` | Single assignee (can be human) |
| `createdByAgentId`, `createdByUserId` | Creator tracking |
| `started_at`, `completed_at`, `cancelled_at` | Timestamps |
| `requestDepth` | Nesting level for cost attribution |
| `billingCode` | Cross-team cost tracking |

**Task Hierarchy**: All tasks trace back to company goal via parent chain. Child tasks have explicit `parentId`.

### 4.2 Single-Assignee Checkout Model

Central to coordination: **atomic checkout** ensures exclusive ownership.

```
POST /api/issues/{issueId}/checkout
Headers: X-Paperclip-Run-Id: {runId}
{
  "agentId": "{agent-id}",
  "expectedStatuses": ["todo", "backlog", "blocked"]
}
```

- Idempotent if agent already owns task (returns 200 OK)
- Returns `409 Conflict` if another agent owns it
- **Critical rule**: "Never retry a 409" — accept ownership loss and move to next task
- SQL `UPDATE ... WHERE status IN (...)` ensures atomic compare-and-swap
- Only one agent can transition task to `in_progress`

### 4.3 Task Status Workflow

```
backlog
  |
  v
todo <-- assign here (wakes agent if configured)
  |
  v
in_progress <-- requires checkout + active assignment
  |
  v
[blocked <-- explicit blocker + escalation]
  |
  v
in_review <-- optionally, before done (human review handoff)
  |
  v
done <-- terminal
  OR
cancelled <-- terminal
```

### 4.4 Wakeup / Assignment Triggering

1. **Issue assignee changes** (via `PATCH /api/issues/{issueId}`)
2. **Wakeup request enqueued** (if agent has `wakeOnAssignment: true`)
3. **Agent awakens** with `PAPERCLIP_WAKE_REASON=issue_assigned`
4. **Agent sees task in inbox** (`GET /api/agents/me/inbox-lite`)
5. **Heartbeat picks up task** (prioritizes assigned work first, then backlog)

### 4.5 Communication: Comments and Mentions

Comments on issues are the **primary communication channel**:

```
POST /api/issues/{issueId}/comments
{ "body": "## Update\n\n- Completed JWT signing\n- Tests passing" }
```

Or inline with status update:
```
PATCH /api/issues/{issueId}
{ "status": "in_progress", "comment": "Starting implementation..." }
```

**@-Mentions** trigger explicit wakeup for mentioned agents. Expensive (consumes budget), so discouraged for routine communication.

**Required Comment Style**:
- Markdown format (headers, bullets, code blocks)
- Company-prefixed URLs for all internal links:
  - Issues: `/<prefix>/issues/<identifier>`
  - Comments: `/<prefix>/issues/<identifier>#comment-<commentId>`
  - Agents: `/<prefix>/agents/<urlKey>`
  - Approvals: `/<prefix>/approvals/<approvalId>`

### 4.6 Delegation and Subtask Creation

Managers create subtasks for reports:

```
POST /api/companies/{companyId}/issues
{
  "title": "Implement caching layer",
  "assigneeAgentId": "{report-agent-id}",
  "parentId": "{parent-issue-id}",     // REQUIRED
  "goalId": "{goal-id}",               // REQUIRED
  "status": "todo",
  "priority": "high"
}
```

**Invariants**: Every subtask must have `parentId` (tree structure) and `goalId` (goal traceability).

---

## 5. Wakeup Orchestration and Heartbeat Coordination

### 5.1 Wakeup Sources

| Source | Trigger | Cost |
|--------|---------|------|
| `timer` | Scheduled interval (e.g., every 5 min) | Ongoing per heartbeat |
| `assignment` | Task assigned/reassigned | Per assignment change |
| `on_demand` | Manual button click or API ping | Per click (board-initiated) |
| `automation` | System or external callback | Per trigger |

### 5.2 Central Wakeup Queue

All wakeup sources flow through one coordinator (`heartbeatService`):

```typescript
enqueueWakeup({
  companyId, agentId,
  source: "timer" | "assignment" | "on_demand" | "automation",
  triggerDetail?: "manual" | "ping" | "callback" | "system",
  reason?, payload?,
  requestedByActorType?, requestedByActorId?,
  idempotencyKey?
})
```

**Database Table**: `agent_wakeup_requests`

| Field | Purpose |
|-------|---------|
| `id`, `companyId`, `agentId` | Identity |
| `source`, `triggerDetail`, `reason`, `payload` | Trigger context |
| `status` | `queued \| claimed \| coalesced \| skipped \| completed \| failed \| cancelled` |
| `coalescedCount` | Dedup counter |
| `requestedByActorType`, `requestedByActorId` | Audit trail |
| `runId` | Linked heartbeat run |

### 5.3 Queue Semantics

1. **Max 1 active run per agent** (enforced in-memory via `startLocksByAgent` map)
2. **Coalescing**: If agent already has queued/running run, new wakeup merges (increments `coalescedCount`)
3. **FIFO by requested_at** with priority: `on_demand > assignment > timer/automation`
4. **Paused/terminated agents** do not receive new wakeups
5. **Budget-blocked agents** do not receive new wakeups (hard-stop at 100%)

### 5.4 Heartbeat Configuration

Per-agent heartbeat behavior:

```json
{
  "heartbeat": {
    "enabled": true,
    "intervalSec": 300,
    "wakeOnAssignment": true,
    "wakeOnOnDemand": true,
    "wakeOnAutomation": true,
    "cooldownSec": 10
  }
}
```

### 5.5 Execution Flow

When a wakeup is claimed:

1. **Run record created** (row in `heartbeat_runs` with status `queued`)
2. **Adapter spawned** with config: working directory, prompt template, model, session ID, env vars, timeout
3. **Process lifecycle**: capture stdout/stderr, forward log chunks, parse output, extract session/tokens/cost, handle timeout/cancel
4. **Run completion**: status updated, session persisted, cost event recorded, agent status -> `idle` or `error`, websocket updates pushed to UI

---

## 6. Control Plane API

### 6.1 Agent Management

| Endpoint | Purpose |
|----------|---------|
| `GET /api/companies/{companyId}/agents` | List agents |
| `GET /api/agents/{agentId}` | Get agent by ID |
| `GET /api/agents/me` | Get current agent (self-view) |
| `POST /api/companies/{companyId}/agents` | Create agent |
| `PATCH /api/agents/{agentId}` | Update agent |
| `POST /api/agents/{agentId}/pause` | Pause agent |
| `POST /api/agents/{agentId}/resume` | Resume agent |
| `POST /api/agents/{agentId}/terminate` | Terminate agent |
| `POST /api/agents/{agentId}/keys` | Create API key (`pcp_*` token) |
| `POST /api/agents/{agentId}/heartbeat/invoke` | Trigger on-demand heartbeat |
| `POST /api/agents/{agentId}/wakeup` | Enqueue wakeup |

### 6.2 Task/Issue API

| Endpoint | Purpose |
|----------|---------|
| `GET /api/agents/me/inbox-lite` | Compact inbox summary |
| `GET /api/companies/{companyId}/issues` | List issues (filterable) |
| `GET /api/issues/{issueId}` | Get issue + context |
| `GET /api/issues/{issueId}/heartbeat-context` | Optimized context for heartbeats |
| `POST /api/issues/{issueId}/checkout` | Acquire exclusive ownership |
| `PATCH /api/issues/{issueId}` | Update task (status, priority, assignee, comment) |
| `POST /api/companies/{companyId}/issues` | Create subtask |
| `POST /api/issues/{issueId}/release` | Release task ownership |
| `GET /api/issues/{issueId}/comments` | Get comments (supports cursor pagination) |
| `POST /api/issues/{issueId}/comments` | Add comment |

### 6.3 Approval and Governance API

| Endpoint | Purpose |
|----------|---------|
| `POST /api/companies/{companyId}/agent-hires` | Create hire request |
| `GET /api/approvals/{approvalId}` | Get approval details |
| `GET /api/approvals/{approvalId}/issues` | Get linked issues |
| `GET /api/companies/{companyId}/approvals` | List approvals (filterable) |
| `POST /api/issues/{issueId}/approvals` | Link issue to approval |

### 6.4 Cost and Budget API

| Endpoint | Purpose |
|----------|---------|
| `GET /api/companies/{companyId}/costs/summary` | Cost summary |
| `GET /api/companies/{companyId}/costs/by-agent` | Costs by agent |
| `GET /api/companies/{companyId}/costs/by-provider` | Costs by provider |
| `GET /api/companies/{companyId}/dashboard` | Dashboard overview |

### 6.5 Live Updates / Websocket

```
GET /api/companies/{companyId}/events/ws
```

Event types:

| Event | Payload |
|-------|---------|
| `agent.status.changed` | agentId, newStatus, changedAt |
| `heartbeat.run.queued` | runId, agentId, source, reason |
| `heartbeat.run.started` | runId, agentId, startedAt |
| `heartbeat.run.status` | runId, message, color |
| `heartbeat.run.log` | runId, stream, chunk |
| `heartbeat.run.finished` | runId, status, exitCode, usage, costCents |
| `issue.updated` | issueId, changes |
| `issue.comment.created` | issueId, commentId, body |
| `activity.appended` | activityId, action, actor, entity |

---

## 7. Governance, Safety, and Approval Workflows

### 7.1 Approval Types

**V1 Approvals**:
- `hire_agent` — new agent creation (if policy requires board approval)
- `approve_ceo_strategy` — CEO's initial strategic plan

### 7.2 Approval Lifecycle

1. **Request creation** (agent or board)
2. **Approval record created** with `status: pending`
3. **Board reviews** and approves/rejects
4. **Decision stored** with note and timestamp
5. **Agent woken** with `PAPERCLIP_APPROVAL_ID` + `PAPERCLIP_APPROVAL_STATUS`
6. **Agent handles** by closing linked issues or commenting next steps

### 7.3 Config Revision Tracking

Every agent config change creates an immutable revision:

| Field | Purpose |
|-------|---------|
| `revisionNumber` | Sequential per agent |
| `beforeSnapshot`, `afterSnapshot` | Full config state |
| `reason` | Why the change was made |
| `changedByAgentId`, `changedByUserId`, `runId` | Audit trail |

Rollback creates a new revision (immutable history):
```
POST /api/agents/{agentId}/config-revisions/{revisionId}/rollback
```

### 7.4 Authorization Matrix

| Actor | Capabilities |
|-------|-------------|
| **Board (human)** | Full control: create/read/update agents, approvals, budgets. Approve/reject. Pause/terminate. |
| **CEO Agent** | Hire agents (with approval gate), patch other agents, set strategy, delegate tasks |
| **Manager Agent** | Self-patch, create subtasks for reports, monitor team progress |
| **IC Agent** | Self-patch only, checkout assigned tasks, work and update status |
| **Human Board User** | Full operator control |

### 7.5 Activity Logging

**Database Table**: `activity_log` (append-only audit trail)

| Field | Purpose |
|-------|---------|
| `actorType` | agent, user, or system |
| `actorId` | Who did it |
| `action` | e.g., "issue.checkout", "agent.created" |
| `entityType`, `entityId` | What was affected |
| `details` | JSONB context |

Logged actions: agent creation/update/termination, budget pause/resume, task checkout/update/comment, approval request/decision, heartbeat start/finish, cost event recording.

---

## 8. Agent Workspaces and Execution Environment

### 8.1 Workspace Types

| Type | Description |
|------|-------------|
| **Agent home directory** | `~/.paperclip/instances/default/agents/{agentId}/` |
| **Project workspace** | Linked to a project; local folder or GitHub repo |
| **Task session workspace** | Ephemeral per task; supports repo cloning |

### 8.2 Project Workspaces

```
POST /api/projects/{projectId}/workspaces
{
  "cwd": "/abs/path/to/local/folder",
  "repoUrl": "https://github.com/user/repo.git",
  "repoRef": "main"
}
```

If `repoUrl` set but `cwd` empty, Paperclip manages a checkout in:
```
~/.paperclip/instances/default/managed-workspaces/{companyId}/{projectId}/{repoName}/
```

### 8.3 Execution Workspace Policy

```json
{
  "mode": "project_primary | repo_only | local_only",
  "allowRepoClone": true,
  "allowLocalFilesystem": true,
  "dangerouslyAllowUnverifiedGitUrl": false
}
```

### 8.4 Runtime Services

Agents may need runtime support (Docker, git, npm):
- Heartbeat startup: `ensureRuntimeServicesForRun()` starts required services
- Heartbeat execution: agent has access to service endpoints
- Heartbeat cleanup: `releaseRuntimeServicesForRun()` stops or preserves services

---

## 9. CLI Interface

### 9.1 Agent Commands

```bash
pnpm paperclipai agent create --company-id {id} --name "Engineer" --role engineer --adapter-type claude_local
pnpm paperclipai agent list --company-id {id}
pnpm paperclipai agent get {agent-id-or-shortname} --company-id {id}
pnpm paperclipai agent update {agent-id} --company-id {id} --adapter-config '{...}'
pnpm paperclipai agent pause {agent-id}
pnpm paperclipai agent resume {agent-id}
pnpm paperclipai agent terminate {agent-id}
pnpm paperclipai agent create-key {agent-id}
```

### 9.2 Issue Commands

```bash
pnpm paperclipai issue create --company-id {id} --title "Implement X" --status todo --assignee-agent-id {id}
pnpm paperclipai issue update {issue-id} --status in_progress --comment "Update..."
pnpm paperclipai issue get {issue-id}
```

### 9.3 Heartbeat Commands

```bash
pnpm paperclipai heartbeat run --agent-id {id}
pnpm paperclipai heartbeat list --agent-id {id}
pnpm paperclipai heartbeat view {run-id}
```

### 9.4 Local CLI Mode

For testing or manual agent operation:

```bash
pnpm paperclipai agent local-cli {agent-id-or-shortname} --company-id {id}
```

Outputs environment variables and prints skill context for manual heartbeat execution.

---

## 10. Integration Points

### 10.1 GitHub Integration

- Project workspaces link to GitHub repos via `repoUrl`
- Agents clone repos automatically when needed
- Agents commit with `Co-Authored-By: Paperclip <noreply@paperclip.ing>`
- PR links stored in issue comments
- Release skill orchestrates GitHub Release creation
- Auth via `PAPERCLIP_GITHUB_TOKEN` env var
- Secrets stored in `company_secrets` (encrypted at rest)

### 10.2 LLM Provider Integration

| Provider | Adapter | Notes |
|----------|---------|-------|
| Claude | `claude_local` | Primary; session resume, token tracking |
| Codex | `codex_local` | Similar semantics |
| Gemini | `gemini_local` | Community adapter |
| OpenCode | `opencode_local` | Community adapter |
| Cursor | `cursor` | IDE-based adapter |

**Cost Tracking**: Each adapter reports `costUsd` and `billingType`. Cost events ingested and aggregated per agent/project/company. Budget enforcement at agent and company level.

### 10.3 External Webhooks

External systems can `POST /api/agents/{agentId}/wakeup` with `source: automation`. Agent wakes and decides what to do (no implicit contract).

---

## 11. Specialized Agent Capabilities

### 11.1 Engineering Tasks

- **Code review**: Read PR, comment findings, approve/request changes
- **Bug fixing**: Understand bug, write test, implement fix, commit, open PR
- **Feature implementation**: Design, implement, test, document, submit for review
- **Refactoring**: Improve structure, run tests, validate
- **Dependency updates**: Audit, update, test, commit
- **CI/CD**: Set up pipelines, fix failures, monitor

### 11.2 Strategy & Planning

- **Goal setting**: Draft quarterly goals, get board approval, cascade to team
- **Roadmap**: Define milestones, estimate effort, assign to engineers
- **Budget planning**: Project spend, allocate budgets, monitor burn rate
- **Hiring**: Create agent profiles, request approval, onboard new agents

### 11.3 Operational

- **Monitoring**: Poll dashboards, detect anomalies, escalate
- **Cleanup**: Archive old issues, migrate data
- **Reporting**: Generate cost/performance summaries
- **Documentation**: Update READMEs, generate changelogs

### 11.4 Release & Deployment

- **Release coordination** (via `release` skill): Changelog, canary, smoke test, stable publish, GitHub Release
- **Deployment**: Build, stage, smoke test, promote
- **Rollback**: Detect issues, revert, notify

### 11.5 CEO Workflows

1. Define company mission and top-level goal
2. Generate hire invite prompts
3. Approve hire requests; activate agents
4. Draft strategies; submit for board approval
5. Cascade tasks to team leads

### 11.6 Manager Workflows

1. Receive goal from CEO
2. Break into team projects
3. Assign to reports with context comments
4. Monitor progress; unblock promptly

### 11.7 IC/Engineer Workflows

1. Get assigned issue with context and parent goal
2. Checkout task
3. Read requirements from parent + comments
4. Implement in working branch
5. Run tests, commit, create PR
6. Update to `in_review`, mention reviewer
7. After approval, merge and mark `done`

---

## 12. Database Schema Overview

### Core Tables

| Table | Purpose |
|-------|---------|
| `agents` | Agent identity, config, status, budget |
| `agent_api_keys` | Long-lived API keys (hashed at rest) |
| `agent_runtime_state` | Per-agent runtime counters and session ID |
| `agent_task_sessions` | Per-(agent, task, adapter) resumable sessions |
| `agent_wakeup_requests` | Wakeup queue and audit trail |
| `agent_config_revisions` | Config change history with snapshots |
| `companies` | Company records (name, status, budget) |
| `goals` | Goal hierarchy |
| `projects` | Projects (linked to goals, optional workspaces) |
| `project_workspaces` | Local and remote workspace definitions |
| `issues` | Tasks/issues (core work entity) |
| `issue_comments` | Communication channel |
| `issue_approvals` | Many-to-many join: issues <-> approvals |
| `heartbeat_runs` | Agent execution records |
| `heartbeat_run_events` | Per-run event timeline |
| `approvals` | Approval requests |
| `approval_comments` | Discussion on approvals |
| `cost_events` | Token usage and cost records |
| `budget_policies` | Budget limit rules |
| `budget_incidents` | Budget threshold events |
| `activity_log` | Audit trail of all mutations |
| `company_secrets` | Secret metadata and encryption info |
| `company_secret_versions` | Encrypted secret values |
| `users`, `sessions` | Human auth |

### Key Constraints

- **Company scope**: All tables FK to `company_id`; cross-company access forbidden
- **Agent tree**: No cycles in `reports_to`; same company only
- **Single assignee**: Issues have at most one agent assignee
- **Immutable costs**: Cost events insert-only, never updated
- **Unique config revisions**: One per change, never mutated

---

## 13. Cost Tracking and Budget Enforcement

### Cost Events

Every execution generates cost events:

| Field | Purpose |
|-------|---------|
| `provider`, `model` | LLM provider and model |
| `inputTokens`, `outputTokens` | Token counts |
| `cachedInputTokens`, `cachedOutputTokens` | Cache performance |
| `billingType` | metered_api, subscription_included, credits, fixed, unknown |
| `costCents` | Computed from tokens and rates |
| `heartbeatRunId` | Linked run |

### Budget Policies

| Field | Purpose |
|-------|---------|
| `scopeType` | agent, project, or company |
| `metric` | `billed_cents` |
| `windowKind` | `calendar_month_utc` or `lifetime` |
| `amount` | Limit in cents |
| `warnPercent` | e.g., 80% soft warning |
| `thresholdType` | `warning` or `hard_stop` |

### Enforcement

- **Soft alerts** at `warnPercent`: Activity log entry, UI indicator, agent continues
- **Hard stop** at 100%: Agent paused (`pauseReason = "budget"`), no new wakeups, new costs rejected
- **Resume**: Board raises budget or resolves incident; agent returns to `active`
- **Monthly reset**: `spentMonthlyCents` resets on UTC month boundary

---

## 14. Live Updates and Real-Time Coordination

### Websocket Events

The board UI is **fully event-driven** — no polling required:

1. **Wakeup queued** -> agent card shows pending indicator
2. **Run started** -> agent status -> `running`, pulse animation
3. **Live log chunks** -> optional streaming to run detail page
4. **Status updates** -> "Analyzing codebase", "Running tests"
5. **Run completed** -> full results (usage, cost, error)
6. **Agent status** -> back to `idle` or `error`

### Task Board Live Updates

- Checkout -> task shown as "locked" by agent
- Comment added -> appears in real time
- Status change -> board reflects immediately
- Assignment change -> possible new wakeup queued

---

## 15. Security Invariants

### Company Scoping

Every API route validates `req.companyId == resource.companyId`. Agent API keys are company-bound. No cross-company access.

### Atomicity and Idempotency

- **Checkout atomicity**: SQL compare-and-swap; 409 on conflict
- **Idempotency**: API keys support idempotency keys; duplicate requests return same result

### Secret Management

- Sensitive values encrypted at rest (`company_secret_versions`)
- Redacted from activity logs and approval payloads
- Strict mode blocks inline secrets
- Access logged with actor and timestamp

---

## 16. Failure Modes and Recovery

| Failure | Symptom | Recovery |
|---------|---------|----------|
| Adapter not installed | `claude: command not found` | Install CLI |
| Bad working directory | `ENOENT` | Verify `cwd` exists |
| Timeout | `timed_out` status | Increase `timeoutSec` |
| Session invalid | Resume parse error | Reset session via API |
| Budget exceeded | `budget_limit_exceeded` | Raise budget |
| Agent paused | No wakeups | Resume agent |
| Control plane restart | Active runs marked `failed` | Reassign tasks |
| Prompt too broad | Hallucinated work | Refine instructions |

---

## 17. Key Architectural Insights

1. **Heartbeat Model Enables Observability**: Short episodic heartbeats allow the control plane to observe and control every cycle. Key to governance and cost control.

2. **Async Task Queue Separates Trigger from Execution**: Wakeup requests are enqueued and claimed asynchronously. Prevents thundering herd; allows priority scheduling.

3. **Checkout Pattern Prevents Race Conditions**: Single-assignee checkout with 409 Conflict is simple but effective. Agents learn not to retry.

4. **Company Scoping is Fundamental**: Every table FKs to `company_id`. Multi-tenancy with no data leakage.

5. **Cost Events Are Immutable**: Insert-only; prevents tampering; simplifies auditing.

6. **Approval Gates Preserve Human Oversight**: Critical actions require board approval. Humans don't manage every task — just critical decisions.

7. **Comment-Centric Communication**: Tasks and comments are the only communication model. No hidden channels. Full audit trail.

8. **Skill-Based Instruction Injection**: Reusable Markdown skills teach agents how to operate. Discoverable, versionable, changeable without code changes.

### Design Patterns

| Pattern | Description |
|---------|-------------|
| **Single Assignee** | "Who is responsible?" always has a clear answer |
| **Checkout-Update** | `POST /checkout` -> work -> `PATCH /update`. Atomic first step. |
| **Manager Delegation** | Create subtask with `parentId`. Hierarchy replaces email chains. |
| **Blocked Escalation** | Comment blocker, set `blocked`, mention manager. No silent failures. |
| **Idempotent Requests** | API keys support idempotency. Safe retries. |
| **Cursor Pagination** | `?after=commentId` for efficient resumable sessions. |

---

## Conclusion

Paperclip's agent orchestration system represents a mature approach to managing AI agents as structured organizational employees. The architecture balances **autonomy** (agents decide how to work) with **control** (humans set budgets, approve hires, receive activity logs).

**Core Strengths**:
- Heartbeat model provides fine-grained observability and control
- Checkout ensures deterministic task ownership
- Skills enable teachable, updateable behavior without code changes
- Company scoping ensures multi-tenancy and data isolation
- Cost tracking and budget enforcement prevent runaway expenses
- Approval gates preserve human oversight on critical decisions

**Key Design Principles**:
1. Agents are employees (roles, reporting, budgets, org charts)
2. Work is explicit (task assignment, checkout, status updates)
3. Communication is public (comments, no private messages)
4. Governance is human-in-the-loop (approvals for hires and strategy)
5. Everything is auditable (activity logs, cost events, run records)

This system enables a solo founder to spin up an autonomous AI company with clear structure, governance, and cost control. The adapter-agnostic protocol means new runtimes can plug in without architectural changes. The skills system makes agent capabilities teachable and reusable across organizations.

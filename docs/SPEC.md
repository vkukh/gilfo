# Gilfo — technical specification (MVP)

Nov 15, 2026 · @Vitalii Kukhar

## 1. Goal and MVP scope

Gilfo takes routine support and development work off the project owner: it watches the logs, investigates incidents, prepares fixes as merge requests, and keeps the backlog. The person keeps three decisions: allow a fix, approve the backlog, accept a MR.

The MVP covers one server and one project. Control goes through a Telegram bot by text and voice messages; replies come back as text.

The minimal platform has four parts: Telegram as the communication and control interface, an admin panel with server settings for each product, AI Server (the orchestrator), and AI Server Node toolkit as its subagent on each server. SDLC roles (Developer, PO, and others) are a separate layer on top of the platform, not its components.

The MVP includes:

- log analysis every 15 minutes over a 20-minute window, and an incident report with a proposed fix;
- a fix after confirmation: branch, commit, GitLab MR, link in Telegram;
- daily log analytics for each service;
- a conversation about the project state at any time;
- a PO agent that keeps the backlog in Linear and takes approved tasks through to a MR;
- safe server actions (restart, rollback) after confirmation.

The target picture is wider than the MVP: Gilfo runs the whole SDLC in the owner's style, from server setup to development and operations, and any project can be paused. This is described in sections 5 and 6 and split into stages in section 12.

Out of MVP scope: an own mobile app, voice replies, several servers, merge without a person.

## 2. Decisions made

| Question | Decision | What it means for the implementation |
| --- | --- | --- |
| Interface | Telegram bot | Text, voice messages, inline decision buttons. Server and product settings are set in the admin panel. A mobile app later connects to the same API |
| Where agents run | Any runtime through AgentRuntime (Cursor SDK first, then OpenCode), locally on the project server | Agents work in a separate working copy of the repository, not in the deploy directory. Isolation and resource limits are required (section 9) |
| Infrastructure | Docker Compose, systemd, PM2 | Three log adapters: Docker, journald, PM2 files |
| Production actions | Only from the runbook and after confirmation | The agent proposes an action; deterministic AI Admin code executes it |
| Git | GitLab | Branch and MR through the GitLab API, statuses through webhooks |
| Backlog | Linear | The PO agent creates and moves tasks; incidents get a task too |
| PO agent role | Proposes the backlog, the owner approves | Development starts only for approved tasks |
| Starting scale | 1 server, 1 project | The protocol and data model are multi-server from the start |
| Coverage | The whole SDLC in the owner's style | A playbook with the stack, conventions, and templates is required (section 5) |
| Pause | A project can be stopped and resumed | Project state in Core, a context snapshot on stop (section 6) |

## 3. Architecture

The platform has four parts: the Telegram bot, the admin panel, AI Server (the orchestrator), and AI Server Node toolkit on each server. This is the minimal set; the SDLC roles from section 4 work on top of it and are not part of the platform.

&#91;embedded content: Gilfo architecture · Core, AI Admin, agents, external systems\]

The Node opens the connection to AI Server itself, so the product server needs no inbound ports. Models run in the provider's cloud even in local mode, so the server needs outbound internet access.

Further in this document, "Core" means AI Server and "AI Admin" means AI Server Node.

### 3.1. Telegram

The only communication and control interface: questions by text and voice, notifications, reports, action confirmation with buttons. Details are in section 8.

### 3.2. Admin panel

A web interface for settings that are inconvenient and unsafe to set in a chat. The panel only stores configuration: it has no server actions; those go through Telegram with confirmation.

For each product it sets:

- servers: IP, SSH port, environment (prod, staging);
- users: accounts on the server (who to connect and run services as) and who has access to the product in Telegram;
- variables: the product's environment variables and Gilfo parameters (schedule, quiet hours, spending limits, model for each role);
- keys: SSH keys, GitLab and Linear tokens, the model provider key;
- services: what to monitor, log sources, health checks.

The panel stores keys to all servers, so it is the most valuable target in the system. Mandatory requirements:

- secrets are encrypted in the database; the encryption key is stored outside the database;
- after saving, a secret is shown only masked and cannot be read back;
- login with two-factor authentication; the panel is not reachable from the open internet (VPN or an IP allowlist);
- every settings change goes into the audit log;
- a Node receives only its own product's secrets; agents do not see them.

### 3.3. AI Server (orchestrator)

The central agent the owner talks to. It reads configuration from the admin panel, holds state, and decides what to run and on which server. Each server's Node toolkit is connected to it as a subagent.

AI Server is hosted separately from the product servers (a small VPS). Otherwise a server failure would also silence the notifications. Modules:

- Dialog Router: detects intent and product, answers itself or calls the tools of the right Node.
- Scheduler: 15-minute checks, daily report, backlog task cycle.
- Incident Manager: incident lifecycle, deduplication, link to the MR and the Linear task.
- Approval Service: every state-changing action waits for the owner's decision; a request has an expiry.
- Integrations: Telegram, GitLab, and Linear (API, webhooks).
- Storage, audit log, cost accounting.

### 3.4. AI Server Node toolkit

A lightweight daemon on the product server (a systemd service under a separate user). The Node makes no decisions: it gives the orchestrator a set of tools and executes their calls.

| Tool | What it does | Owner confirmation |
| --- | --- | --- |
| `logs.query` | Returns log entries for a window after the pre-filter and secret masking | Not required |
| `services.status` | Service state, disk, memory, health checks | Not required |
| `repo.read` | Reads code and git history in the working copy | Not required |
| `agent.run` | Runs a coding agent through the selected runtime in a separate worktree with a role from the playbook | Required for code changes |
| `mr.create` | Pushes a `gilfo/*` branch and creates a GitLab MR | Part of a confirmed fix or task |
| `runbook.execute` | A runbook action: restart, rollback, disk cleanup, Ansible role | Always required |

Locally the Node keeps only the log pre-filter, which works without a model, and an event buffer in case the connection drops. A new tool is added to the toolkit with no changes to the orchestrator.

Installation: the orchestrator connects to the server over SSH with the IP, user, and key from the admin panel and installs the Node. After that, all work goes through the connection the Node opens itself, and the orchestrator's SSH access can be revoked.

AI Server ↔ Node protocol: WebSocket over TLS, a token per Node, messages with identifiers for idempotency. If the connection is lost, the Node buffers events locally, and after 3 missed heartbeats AI Server sends a "server unreachable" notification.

## 4. SDLC roles (layer above the platform)

Roles are not platform components. A role is configuration in the playbook: a system prompt, a set of Node tools, and permissions. The orchestrator picks a role for the task and passes it to `agent.run`, so a role can be added or changed with no changes to the platform code.

Permissions are minimal for the role: whoever reads logs cannot write code or run commands. The set of roles below is a starting point.

| Agent | What it does | Permissions | Result |
| --- | --- | --- | --- |
| Log Analyst | Classifies selected log entries: incident or noise, severity, affected service | Read only: logs, aggregates, code | Incident card, daily report text |
| Developer | Finds the cause in the code, proposes a fix, and after a go-ahead implements the fix or a backlog task | Writes only to the task worktree; tests and linter in the sandbox; push only to `gilfo/*` branches | Fix proposal, branch, MR |
| PO | Turns the owner's goals into tasks with acceptance criteria, prioritizes, tracks progress | Reads code; writes to Linear | Backlog for approval, weekly summary |
| Architect | Designs solutions for new projects and large tasks, checks compliance with the playbook | Reads code; writes only to `docs/adr` | ADR, technical plan for the task |
| Reviewer | Checks a MR before showing it to the owner: diff, tests, conventions, security | Read only | MR comments, verdict |
| DevOps | Prepares the server and CI/CD from templates, proposes runbook actions during incidents | Writes infrastructure code to the repository; executes nothing on the server itself | MR with infrastructure code, action request |

Two rules apply to everyone. First: server changes are made not by the model but by the Action Executor from a runbook template. Second: text from logs and external sources is data for the agent, not instructions.

## 5. Full SDLC in the owner's style

The "style" is set by the playbook: a separate `gilfo-playbook` repository that is attached to every agent run. Without it, agents would write "average" code, not the owner's code.

The playbook contains:

- the stack and project templates (repository skeleton, directory structure, base dependencies);
- code conventions, naming, commit and MR rules as `.cursor/rules`;
- Definition of Done: what a MR must contain, which tests are mandatory;
- infrastructure templates: Ansible roles for the server, Docker Compose, reverse proxy config, GitLab CI;
- the runbook of allowed server actions;
- templates for tasks, ADRs, and reports.

Gilfo builds the first version of the playbook itself: the Architect analyzes the owner's existing repositories, writes down the conventions it finds, and hands them over for editing and approval. After that, every owner comment in a MR that repeats becomes a proposal for a new rule.

| Stage | Who leads | What happens | Owner decision |
| --- | --- | --- | --- |
| Idea | PO | A voice or text description becomes a short product spec and backlog | Approve the spec and backlog |
| Architecture | Architect | Picking a template from the playbook, ADR, service and data diagram | Approve the ADR |
| Repository setup | Developer, DevOps | GitLab repository, code skeleton, CI, branch protection, Linear project | Accept the first MR |
| Server setup | DevOps, AI Admin | One command installs AI Admin; then it applies Ansible roles: users, firewall, Docker, reverse proxy, TLS, backups | Confirm the change plan before applying |
| Development | PO, Developer | Approved tasks go into work one at a time, each ending with a MR | None |
| Review | Reviewer | Automatic MR check, fixing comments, green CI | Accept or return the MR |
| Release | DevOps, AI Admin | Deploy through GitLab CI after merge; health-check verification; rollback by runbook | Confirm the production deploy |
| Operations | Log Analyst, Developer | Log monitoring, incidents, fixes, daily report | Allow a fix or an action |
| Maintenance | Developer | Dependency updates, vulnerabilities, technical debt as backlog tasks | Approve tasks |

Server setup is declarative: the agent picks roles and variables, Ansible first shows the change plan (`--check --diff`), the owner confirms, and only then are the roles applied. There are no arbitrary root commands from the model.

## 6. Project pause

A project has a state that the owner changes with a command in Telegram. A pause stops spending and agent work but keeps everything needed to continue.

| State | Development (PO, Developer) | Monitoring and reports | When to use |
| --- | --- | --- | --- |
| `active` | Running | Running | Normal mode |
| `dev_paused` | Stopped | Running | The product is live, new features are not needed |
| `paused` | Stopped | Only heartbeat and health checks | The project is frozen |
| `archived` | Stopped | Stopped, AI Admin disconnected | The project is closed |

During a pause, Gilfo:

1. Stops taking new tasks and turns off the project schedule.
2. Lets the active agent finish its current step, or cancels the run if the owner chose "stop now".
3. Pushes unfinished work to a branch marked WIP and turns the MR into a draft.
4. Writes a snapshot: what is done, what is in progress, open incidents and MRs, the next step. The snapshot is stored in Core and in the repository (`docs/gilfo/state.md`).
5. Marks the tasks in Linear as paused.

On resume, the agent reads the snapshot, updates branches from the default branch, checks what changed during the pause, and sends a short plan for continuing for confirmation. Separately, there is a global emergency stop `/stop`: it immediately cancels all agent runs on all projects.

## 7. Scenarios

### 7.1. Log monitoring and an incident

Every 15 minutes AI Admin analyzes the last 20 minutes of logs. The five-minute overlap catches events on window boundaries, and error fingerprints prevent sending the same incident twice.

1. Log Collector gathers entries for the window from all services.
2. A pre-filter without a model selects anything suspicious: error level and above, stack traces, 5xx codes, container restarts, OOM, a jump in error rate against the baseline, sudden silence from a service.
3. Each error gets a fingerprint (normalized message plus the top stack frames). Known and muted fingerprints go no further.
4. If there is nothing new, the cycle ends without calling a model. This is the main cost lever: 96 cycles a day per server.
5. Otherwise the Log Analyst receives the selected entries with secrets masked and decides: incident or noise, severity, service, hypothesis.
6. For an incident, the Developer in read-only mode analyzes the code and recent commits: cause, fix plan, files, risk, confidence.
7. Core sends an incident report to Telegram with buttons: "Fix", "Details", "Ignore", "Mute similar", and a runbook action when needed ("Restart service").

Severity drives notification behavior: critical ones arrive immediately at any time, medium ones wait for the end of quiet hours, low ones go only into the daily report.

### 7.2. Fix and merge request

1. The owner presses "Fix". Core records the confirmation in the audit log.
2. Workspace Manager creates a worktree and a `gilfo/fix/INC-<number>-<short-description>` branch from the current default branch.
3. The Developer implements the fix, adds a test that reproduces the error, runs the tests and the linter.
4. The Reviewer checks the diff; comments go back to the Developer, no more than 2 rounds.
5. AI Admin pushes the branch and creates a MR through the GitLab API: incident description, cause, what changed, how it was verified, link to the Linear task.
6. The owner gets the MR link and the CI status. The next decision is theirs.
7. The owner's MR comments arrive by webhook; the Developer follows up in the same branch.
8. After merge and deploy, Gilfo watches the fingerprint for 24 hours and either closes the incident or reports that the error came back.

If the tests fail or the agent runs out of time, no MR is created: the owner gets an honest report of what was done and where it got stuck.

### 7.3. Daily report

The report arrives once a day at a set time (09:00 Cyprus time by default). The numbers are computed by code from the aggregates of the 15-minute cycles; the model writes only a short conclusion.

For each service:

- the number of errors and warnings against the previous day and the 7-day average;
- new error fingerprints and the 5 most frequent;
- restarts, downtime, memory and disk peaks;
- incidents for the day and their status.

For the project as a whole: open MRs waiting for review, backlog progress, agent spending for the day.

### 7.4. Conversation about the project state

The owner writes or speaks a question in free form. Dialog Router answers from Core data (incidents, MRs, backlog, spending) and passes questions about the code and the live server state to the agent on AI Admin. Conversation context is kept per project, so follow-up questions need no repetition.

Examples: "how is the project?", "why did the api fail last night?", "what is left in the backlog?", "show me how authorization is done".

### 7.5. Backlog and feature development

1. The owner describes a goal. The PO asks clarifying questions if it cannot do without them.
2. The PO creates tasks in Linear with status Proposed: description, acceptance criteria, size estimate, dependencies.
3. The owner approves the list in Telegram or moves tasks to Approved directly in Linear.
4. The PO takes approved tasks by priority; in the MVP, no more than 1 task per project is in progress at a time.
5. For a large task, the Architect writes a technical plan. The Developer implements it in a `gilfo/feat/<task id>` branch, then as in 7.2.
6. The status in Linear changes automatically: In Progress, In Review after the MR is created, Done after merge.
7. Once a week the PO sends a summary: done, in progress, blocked, proposals for next week.

## 8. Telegram interface

The bot accepts messages only from the owner: the list of allowed Telegram user ids is set in the Core configuration; the bot does not answer anyone else.

- **Text.** Free form or commands.
- **Voice.** A voice message is transcribed to text (Ukrainian, with technical terms in English). The bot shows the transcribed text before answering.
- **Buttons.** Every decision is made with a button, not with text. A voice command never confirms a server action by itself: the bot asks again with a button.
- **Replies.** As text. Long reports are shortened to a summary with a "Details" button.

| Command | Action |
| --- | --- |
| `/status` | Project state: services, open incidents, MRs, tasks in progress |
| `/incidents` | List of incidents and their statuses |
| `/report` | Daily report outside the schedule |
| `/backlog` | Backlog: awaiting approval, in progress, done |
| `/pause`, `/resume` | Pause and resume the project, choosing the pause level |
| `/stop` | Emergency stop of all agents |
| `/budget` | Spending for the day and month, limits |
| `/project` | Choose the active project for the conversation |
| `/new` | Start a new project through the SDLC from section 5 |

An incident report has a fixed structure: severity and service, what happened in one sentence, when and how many times, the likely cause with a link to the file and line, the proposed fix, the fix risk, decision buttons. A confirmation request is valid for 30 minutes for server actions and has no limit for code fixes.

A mobile app will later use the same Core API as Telegram Gateway, so the Gateway holds no business logic.

## 9. Security and permissions

Agents run on the same server as production, so isolation is a mandatory part of the MVP, not an improvement for later. By default, the local Cursor SDK agent runs tools without confirmation, and the sandbox is off.

**Agent isolation**

- A separate `gilfo` user without sudo and without access to the deploy directory and production `.env` files.
- Working copies in `/opt/gilfo/workspaces`; the agent writes only there.
- `sandboxOptions.enabled: true`: commands are restricted to the working directory, network only to an allowlist of addresses (GitLab, package registries).
- Allowed and denied tool lists for each role; `.cursor/hooks.json` hooks block dangerous shell commands.
- CPU and memory limits through a systemd slice, so builds and tests do not take resources from production services.
- Docker access only through a socket proxy that allows reading logs and the container list. Direct socket access is equivalent to root.

**Server actions**

The runbook is a file in the playbook. Each action has a command template, a precondition, a post-check, and a timeout. Parameters from the model are validated against an allowlist of values.

| Action | When it is proposed | Post-check |
| --- | --- | --- |
| Service restart | The service does not respond or is stuck in a restart loop | Health check within 2 minutes |
| Rollback to the previous release | Errors started after a deploy | Health check, the error fingerprint disappears |
| Disk cleanup | Less than 10% free space | Free space after cleanup |
| Applying an Ansible role | Server setup, configuration change | A repeat run in check mode shows no changes |

Actions are executed by a separate Action Executor process with narrow sudo rules for specific commands. The agent has no access to it.

**GitLab and secrets**

- A project access token with rights to push to `gilfo/*` branches and create MRs. The default branch is protected; the token has no merge rights.
- Cursor, GitLab, Linear, and Telegram keys are stored outside the working copies and passed to the process, not to the agent.
- The pre-filter masks tokens, passwords, emails, and phone numbers in logs before the text reaches the model.

**Control**

- Audit log: every agent run, command, confirmation, server action, with time and result.
- Daily and monthly spending limits per project; once exhausted, only monitoring without a model keeps working.
- Limits on agent run time and on the number of MRs per day.
- Emergency stop `/stop`.

## 10. Agent runtime: provider-agnostic

Gilfo is not tied to one agent provider. All access to agents goes through the `AgentRuntime` interface, and a concrete runtime is an adapter chosen in the admin panel for each product. The first adapter is the Cursor SDK; the second is OpenCode, which works with many model providers and with local models.

The contract each adapter implements:

```typescript
interface AgentRuntime {
  id: string;                                   // 'cursor' | 'opencode' | ...
  capabilities(): RuntimeCapabilities;          // what the adapter supports natively
  start(task: AgentTask): Promise<AgentRun>;
  resume(runId: string, message: string): Promise<AgentRun>;
  cancel(runId: string): Promise<void>;
  listModels(): Promise<ModelInfo[]>;
}

interface AgentTask {
  role: RoleConfig;           // prompt, tool policy, model — from the playbook
  workdir: string;            // the task's git worktree
  prompt: string;
  tools: McpServerRef[];      // Gilfo tools as an MCP server
  outputSchema?: JSONSchema;  // incident card, fix plan
  limits: { timeoutSec: number; maxCostUsd: number };
}

interface AgentRun {
  id: string;
  events(): AsyncIterable<AgentEvent>;  // text | tool_call | tool_result | status | usage | error
  result(): Promise<AgentResult>;       // summary, changed files, structured output, cost
}
```

Three rules keep the abstraction honest:

1. **Policy is described once, in Gilfo's terms.** A role in the playbook sets the allowed tools, permissions, and model; the adapter translates this into the runtime's native format.
2. **What the runtime cannot do, Gilfo does.** Gilfo does not hand isolation, structured output validation, and cost accounting to the runtime unless the runtime provides them natively.
3. **Gilfo tools are packaged as an MCP server.** Requesting a runbook action, reading log aggregates, and working with Linear are available to any runtime that supports MCP, with no separate code for each.

| Capability | Cursor SDK | OpenCode | What Gilfo does |
| --- | --- | --- | --- |
| Models | Cursor model catalog, Cursor key only | `provider/model` format, 75+ providers and local models | The model for a role is set in the admin panel; the adapter translates the name |
| Keys | One Cursor API key | Provider keys in the OpenCode config or through `auth.set` | Keeps them in the secret store and passes them to the adapter, not to the agent |
| Launch | `Agent.create`, `agent.send` | `createOpencode` starts a local server, then `session.prompt` | The adapter hides the difference behind `start` and `resume` |
| Event stream | `run.stream()` | `event.subscribe()` (SSE) | Normalizes everything into `AgentEvent` |
| Tool permissions | `tools`, `disallowedTools`, file hooks | `permission`: allow, ask, deny, with rules for individual commands and agents | One role policy, translated by the adapter |
| Isolation | `sandboxOptions` (bubblewrap on Linux) | No separate OS sandbox found in the docs | For OpenCode, runs the process in its own isolation (bubblewrap or a container) |
| Custom tools | `customTools`, MCP | MCP | One MCP server with Gilfo tools |
| Structured output | Not found in the docs | Yes | Validates JSON Schema itself and retries the request on failure |
| Cost accounting | `usage` events | Not checked | Own accounting; where the runtime does not report a price, computes it from tokens and the price list |

An adapter counts as ready only when it passes the shared set of contract tests: read the code and answer, make a fix and pass a test in the worktree, not leave the worktree, cancel correctly, return structured output. The same set runs for each new runtime.

Models are assigned to roles in configuration: a cheap fast model for the Log Analyst and the dialog, a stronger one for the Developer and the Architect. The adapter takes the list of available models from the runtime (`listModels`), because it depends on the account and the provider.

This specification has no cost estimate: it depends on the runtime, the models, and the log volume. Stage 0 measures it on a real project.

## 11. Data model and stack

All long-lived state lives in Core; AI Admin keeps only a local buffer in case the connection drops.

| Entity | Key fields |
| --- | --- |
| Project | name, GitLab repository, Linear team, state (`active`, `dev_paused`, `paused`, `archived`), spending limits, playbook version |
| Server | address, AI Admin token, last heartbeat, Admin version |
| Service | project, server, type (docker, systemd, pm2), log source, health check |
| LogWindow | service, window start and end, counters by level, whether the model was called |
| ErrorFingerprint | fingerprint, first and last occurrence, count, status (new, known, muted) |
| Incident | service, fingerprints, severity, status, cause, Linear task, MR |
| FixProposal | incident, plan, files, risk, confidence |
| Approval | what is being confirmed, decision, time, expiry |
| AgentRun | role, project, task, model, status, duration, cost, link to the run log |
| Action | runbook action, parameters, confirmation, post-check result |
| MergeRequest | project, branch, link, CI status, review status |
| ProjectSnapshot | state snapshot during a pause: done, in progress, next step |
| Conversation | project, messages, agent identifier for resuming |
| AuditEvent | who, what, when, on what, result |

Stack:

- Language: TypeScript in all parts, Node.js 22.13+ (an SDK requirement). Monorepo: `core`, `admin`, `protocol`, `playbook`.
- Core: NestJS, PostgreSQL, task queue and scheduling on pg-boss (no separate Redis), grammY for Telegram.
- Speech recognition: an external STT service or local Whisper; the choice is in the open questions.
- AI Server Node: a Node.js daemon under systemd, runtime adapters (`@cursor/sdk`, `@opencode-ai/sdk`), an MCP server with Gilfo tools, SQLite for the buffer, a WebSocket client.
- Infrastructure code: Ansible, Docker Compose, GitLab CI.
- Admin installation: one script with a one-time registration token that Core issues by a command in Telegram.

## 12. Implementation stages

Each stage gives a useful result on its own. The order goes from the smallest risk to the largest: read first, then code, then server actions.

| Stage | What we do | Acceptance criterion |
| --- | --- | --- |
| 0. SDK check | AgentRuntime interface, contract tests, and the Cursor SDK adapter; a local agent on the server over a working copy: sandbox, hooks, tool limits, cancellation, cost accounting | The agent answers a question about the code; an attempt to write outside the worktree is blocked; the price of one log analysis and one fix is known |
| 1. Skeleton | AI Server, Telegram bot, admin panel, Node toolkit, protocol, `/status`, conversation about the project | Questions by text and voice get an answer from server and repository data; disconnecting the Admin triggers a notification |
| 2. Monitoring | Log collection, pre-filter, 15-minute cycle, incidents with a proposed fix, daily report | An error raised on purpose arrives in Telegram within 20 minutes, once, with a cause in the code; a cycle without errors does not call a model |
| 3. Fix and MR | Implementing the fix after a go-ahead, Reviewer, GitLab MR, Linear task, follow-up on comments, the second runtime adapter (OpenCode) | From the "Fix" button to a MR link with green CI with no manual steps; a push to the default branch is impossible; the same scenario passes on the OpenCode adapter |
| 4. Server actions and pause | Runbook, Action Executor, confirmations, project states, `/pause`, `/resume`, `/stop` | Restart and rollback run only after a button press; a pause stops spending; resume continues from the snapshot |
| 5. PO agent | Backlog in Linear, approval, developing tasks through to a MR, weekly summary | An approved task reaches a MR without the owner; unapproved tasks do not go into work |
| 6. Playbook and full SDLC | Building the playbook from existing repositories, Architect, templates, server setup through Ansible, `/new` | A new project goes from an idea description to a deployed skeleton on a clean server only through confirmations in Telegram |
| 7. Expansion | Several servers and projects, mobile app, voice replies | A second server connects with one command with no changes to Core |

Stage 6 comes after the operational stages on purpose: server setup is the riskiest, and confirmations, audit, and the runbook must already work before it. The pilot of stages 0–5 runs on an existing project and server.

## 13. Risks, open questions, ideas

### Risks

| Risk | How we reduce it |
| --- | --- |
| Agents with write access run on the production server: builds and tests take resources, an isolation failure affects production | Isolation from section 9 and resource limits. Recommendation: as soon as a second server appears, move the Developer to a separate machine or to the SDK cloud mode, leaving only read access on production |
| The agent runtime changes its API or prices (Cursor SDK is still in public beta) | One `AgentRuntime` interface, two adapters (Cursor SDK, OpenCode), shared contract tests, switching the runtime in the admin panel |
| Cost of 96 cycles a day | Pre-filter without a model, fingerprints, spending limits, measurement in stage 0 |
| Logs go to an external model | Masking before sending, Privacy Mode, sending only the selected entries |
| Log text contains instructions for the model (prompt injection) | Log Analyst has no write access; the Developer gets a structured incident card; every change goes through confirmation and MR review |
| A fix treats the symptom or breaks something else | A mandatory test for the error, Reviewer, CI, owner review, 24 hours of watching after deploy |
| A stream of notifications the owner stops reading | Severity levels, quiet hours, muting fingerprints, one incident per cause |

### Open questions

- [ ] What is the pilot project's stack, and does it have tests and GitLab CI?
- [ ] Is GitLab cloud or self-hosted? How is production deployed today?
- [ ] Service log format: JSON or free text? Is there Sentry or other monitoring already?
- [ ] Which Cursor plan, and whose key do we use: personal or a team service account?
- [ ] Monthly spending limit for agents?
- [ ] Speech recognition: external service or local Whisper?
- [ ] Where do we host Core?
- [ ] Time of the daily report and quiet hours?
- [ ] What exactly is "my style": which repositories to take as the reference for the playbook?
- [ ] Is there a staging environment, or do fixes go straight to production after merge?

### Ideas for later

- Incident memory: a base of past causes and fixes that the Developer reads before analysis.
- External availability checks from Core: a service that died silently writes no logs.
- A security agent: vulnerable dependencies, secrets in code, expired certificates.
- Review of MRs the owner writes themselves.
- Release notes and a changelog from merged MRs.
- Real-time voice dialog in the mobile app.
- A web panel with the history of incidents, agent runs, and spending.

## Sources

- [Cursor TypeScript SDK, documentation](https://cursor.com/docs/sdk/typescript.md): facts about the Cursor SDK in section 10.
- [OpenCode SDK](https://opencode.ai/docs/sdk): starting the server, sessions, event stream.
- [OpenCode, permissions](https://opencode.ai/docs/permissions/): allow, ask, deny for tools and agents.
- [OpenHands alternatives: OpenCode overview](https://www.testmuai.com/blog/openhands-alternatives/): 75+ model providers and local models.

As of November 18, 2026.

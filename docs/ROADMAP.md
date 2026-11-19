# Roadmap

Each stage is useful on its own. The order goes from the smallest risk to the largest: read first, then change code, then act on a server.

Stages 0–5 are piloted on one existing project and one server. Stage 6 comes after day-to-day operations on purpose. Setting up a server is the riskiest step, and confirmation, the audit log, and the runbook have to work before it.

## 0. SDK check — Current

`AgentRuntime` interface, shared contract tests, and the Cursor SDK adapter. A local agent on the server, working in a git worktree: sandbox, hooks, tool limits, cancellation, and cost accounting.

**Acceptance:** The agent answers a question about the code. A write outside the worktree is blocked. The cost of one log analysis and one fix is known.

**Where contributors can help:** Review the `AgentRuntime` contract and propose cases for the shared contract tests.

## 1. Skeleton

AI Server, Telegram bot, admin panel, Node toolkit, protocol, `/status`, and a conversation about the project.

**Acceptance:** A question by text or voice gets an answer from server data and the repository. Disconnecting the Node sends a notification.

**Where contributors can help:** Review the Telegram commands and the admin-panel settings list.

## 2. Monitoring

Log collection, a pre-filter, the 15-minute cycle, incidents with a proposed fix, and the daily report.

**Acceptance:** An error raised on purpose arrives in Telegram within 20 minutes, once, with a cause in the code. A cycle with no errors does not call a model.

**Where contributors can help:** Send log samples with secrets removed, and note pre-filter false positives.

## 3. Fix and merge request

The fix after approval, Reviewer, a GitLab merge request, a Linear issue, follow-up on review comments, and the second runtime adapter (OpenCode).

**Acceptance:** From the Fix button to a merge-request link with green CI, with no manual steps. A push to the default branch is impossible. The same scenario passes on the OpenCode adapter.

**Where contributors can help:** Help with the OpenCode adapter once the contract tests exist.

## 4. Server actions and pause

Runbook, Action Executor, confirmation, project states, `/pause`, `/resume`, and `/stop`.

**Acceptance:** Restart and rollback run only after a button press. Pause stops spending. Resume continues from the snapshot.

**Where contributors can help:** Propose a runbook action: command template, precondition, post-check, timeout.

## 5. PO agent

A backlog in Linear, approval, development of tasks through to a merge request, and a weekly summary.

**Acceptance:** An approved task reaches a merge request without the owner taking part in the implementation. Tasks that are not approved do not start.

**Where contributors can help:** Review how Proposed and Approved tasks move in Linear.

## 6. Playbook and full SDLC

A playbook gathered from existing repositories, the Architect role, templates, server setup through Ansible, and `/new`.

**Acceptance:** A new project goes from a description of the idea to a deployed skeleton on a clean server, with confirmations in Telegram as the only input.

**Where contributors can help:** Describe conventions from a real repository that belong in the playbook.

## 7. Expansion

Several servers and projects, a mobile app, and voice replies.

This stage is a step toward running 20–30 projects in parallel.

**Acceptance:** A second server connects with one command and no change to Core.

**Where contributors can help:** Point out what must stay unchanged in Core when a second server is added.

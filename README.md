<p align="center">
  <img src="docs/assets/logo.svg" alt="Gilfo" width="120">
</p>

# Gilfo

AI teammate that lets a small team run 20–30 projects in parallel — you step away from the computer and just talk to it like a colleague.

## In Telegram

Example, not a real incident:

> **You:** why did the api fail last night?
>
> **Gilfo:** Incident, medium — api. Errors started after the last deploy. A fix is ready.
> [Fix] [Details] [Ignore]

You write, or you send a voice message. Gilfo answers in text. A decision is a button.

## Platform

Four parts: a Telegram bot for talk and decisions, an admin panel for servers and product settings, an AI Server (the orchestrator), and an AI Server Node toolkit on each server. SDLC roles such as Developer and PO sit above this platform, outside these four parts.

## What it does

- Checks logs every 15 minutes and reports an incident with a proposed fix.
- Sends a daily report.
- After you confirm, writes the fix and opens a GitLab merge request.
- A PO agent keeps the backlog and takes approved work through to a merge request.
- You can pause a project and resume it later.

## Safety

Agents run in isolation, away from the deploy directory. A person decides when to fix and when to merge. The agent token has no merge rights.

## Runtimes

Gilfo is provider-agnostic. Access goes through an `AgentRuntime` adapter, chosen per product. The first adapter is the Cursor SDK. The second is OpenCode.

## Status

[Stage 0](docs/ROADMAP.md) is current. The start is one server and one project. There is no application code yet. The list above is what Gilfo is being built to do.

## Contributing

Contributions are welcome now, while the repository is still at stage 0. Review the specification, improve the docs, propose runbook actions and playbook ideas, try those ideas on your own servers, or help with translations. The guide is [CONTRIBUTING.md](CONTRIBUTING.md). Small tasks are labeled [good first issue](https://github.com/vkukh/gilfo/issues?q=is%3Aissue+is%3Aopen+label%3A%22good+first+issue%22).

## Docs

- [docs/SPEC.md](docs/SPEC.md) — product and technical specification
- [docs/ROADMAP.md](docs/ROADMAP.md) — stages 0–7 and where to help

## Acknowledgements

People who contribute are listed here.

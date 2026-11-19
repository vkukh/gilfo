# Contributing to Gilfo

## Welcome

Gilfo is an open-source AI teammate for a small team. You talk to it in Telegram, by text or voice, and it helps run many projects in parallel. The project is at stage 0: one server, one project, and no application code yet. Contributions are welcome from the first day. Read [CODE_OF_CONDUCT.md](CODE_OF_CONDUCT.md) before you take part.

## Ways to contribute

There is little code, so the useful work right now is around the spec:

- Feedback on [docs/SPEC.md](docs/SPEC.md): what is unclear, missing, or risky.
- Documentation, including this guide and translations.
- Ideas for runbooks and for the playbook (stack, conventions, templates).
- Testing the ideas on your own servers, and writing down what broke.
- Translations of the docs.

Later, once stage 0 lands, the code to add is a runtime adapter or a tool on the AI Server Node.

## Before you start

Search the [open issues](https://github.com/vkukh/gilfo/issues). If one already covers your change, comment there.

For a large change, open an issue with the proposal first and wait for a maintainer to reply. A large change is a new runtime, a new Node tool, a new runbook action, or anything that edits the architecture rules below.

## Workflow

1. Fork the repository.
2. Create a branch named `feat/…`, `fix/…`, or `docs/…`.
3. Commit with [Conventional Commits](https://www.conventionalcommits.org/): `feat:`, `fix:`, or `docs:`.
4. Open a small pull request and link the issue.
5. Wait until CI is green, then wait for review.

One pull request is one change. There is no CI yet. When stage 0 adds it, a pull request is not ready for review until CI is green.

## Development setup

This section works after stage 0. The commands are not in the repository yet.

- Node.js 22.13 or newer
- pnpm

```bash
pnpm install
pnpm lint
pnpm typecheck
pnpm test
```

## Coding standards

- TypeScript with `strict` enabled.
- Biome for format and lint.
- Vitest for tests.
- Every change includes tests.
- Public types have a short doc comment.

## Architecture rules

These rules come from sections 9 and 10 of [docs/SPEC.md](docs/SPEC.md). A pull request that breaks one of them is not merged.

- Code for a specific runtime lives only in that runtime's adapter.
- A new adapter is ready only when it passes the shared contract tests: read code and answer, apply a fix and pass a test in the worktree, stay inside the worktree, cancel correctly, and return structured output.
- Secrets never reach the agent, the logs, or the repository.
- An action on a server runs only through a runbook, and only after a person confirms it.
- The agent has no merge rights.

## How to add

[docs/SPEC.md](docs/SPEC.md) names the monorepo packages `core`, `admin`, `protocol`, and `playbook`. It does not name the adapter or tool directories. Those paths are TODO until stage 0 creates the packages.

### A runtime adapter

Put the code only in that runtime's adapter, inside the AI Server Node toolkit. Exact directory: TODO.

Implement `AgentRuntime`: `id`, `capabilities`, `start`, `resume`, `cancel`, and `listModels`. The role in the playbook states the allowed tools, rights, and model once. The adapter translates that policy into the runtime's own format. No other package imports that runtime's SDK.

Tests: the shared contract suite listed in Architecture rules. The same suite runs for every runtime.

### A Node tool

Add the tool to the Node toolkit's MCP server. The orchestrator stays unchanged. Exact directory: TODO.

State what the tool does and whether it needs a person's confirmation. Reading logs, service status, or the repository does not. Changing code or the server does.

Tests: the tool's behavior, and a test that a state-changing tool refuses to run without an approval.

### A runbook

Add a file in the separate `gilfo-playbook` repository. Each action has a command template, a precondition, a post-check, and a timeout. Parameters from the model must match an allow-list. Action Executor runs the command. The agent does not. Exact path: TODO until that repository exists.

Until the repository exists, open an issue that contains those four fields.

Tests: TODO until the playbook repository has a test runner.

## AI-assisted contributions

Code written with AI assistance is welcome. The pull request author is responsible for it. You understand the change, you have tested it, and you say so in the pull request description.

## Issues and pull requests

Write issues and pull requests in English. Ukrainian is fine too.

## Reporting bugs

Include:

- Gilfo version or git commit
- Environment (OS, Node.js, runtime if any)
- Steps to reproduce
- Expected result and actual result
- Logs, with secrets, tokens, passwords, emails, and phone numbers removed

Use the bug report template. For a vulnerability, use [SECURITY.md](SECURITY.md) instead of a public issue.

## Security

Do not report a vulnerability in a public issue, pull request, or discussion. Use GitHub private vulnerability reporting, as described in [SECURITY.md](SECURITY.md).

## License

Gilfo is MIT. Contributions are accepted under the same license. See [LICENSE](LICENSE).

## Recognition

Everyone whose contribution is merged is added to the Acknowledgements section in [README.md](README.md).

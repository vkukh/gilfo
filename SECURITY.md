# Security

Gilfo has no release yet. The current work is stage 0 on `main`. Reports about the specification and about this tree are still in scope.

## How to report

Report a vulnerability through GitHub private vulnerability reporting:

<https://github.com/vkukh/gilfo/security/advisories/new>

Do not open a public issue, pull request, or discussion. Do not post exploit details in a commit, a log paste, or a chat.

The maintainer has to turn on private vulnerability reporting in the repository settings (Settings, Code security, Private vulnerability reporting). This file does not change those settings. If the form is unavailable, email kukhar.vitalii@gmail.com. Put "Gilfo security" in the subject and leave the exploit out of the subject line.

Include the git commit or version, the environment, what you expected, what happened, and the impact. Attach logs only after secrets, tokens, passwords, emails, and phone numbers are removed.

## What counts as a vulnerability

- An agent leaves its worktree, or reads the deploy directory or production secrets.
- A secret reaches the agent, a log, or the repository.
- The agent token can merge, or can push to the default branch.
- A server action runs without a runbook, or without a person's confirmation.
- The admin panel is reachable from the open internet, or secrets in it are readable after they are saved.

## What does not count

Ordinary bugs and feature requests belong in a public issue. Use the templates in `.github/ISSUE_TEMPLATE/`.

## Response time

- Acknowledgement within 3 business days.
- An initial assessment within 7 business days.

The assessment says whether the report is accepted, what is affected, and the next step. It does not promise a release date. There is no release yet.

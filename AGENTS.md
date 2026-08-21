# AGENTS.md

`kyriakon` is the meta repo for kyriakon.net — a low-extraction, no-shell hosting platform (mail,
static/Gemini sites, `pass`-compatible git repos) for the Orthodox Christian community. It holds
the one source of truth for decisions and vocabulary that cut across the `kyriakon-*` repos; it holds
no implementation.

## Orientation

This is the cross-cutting repo. The founding project proposal and shared vocabulary live here:

- `docs/decisions/kyriakon-net-project-proposal.md` — the founding proposal; read it before
non-trivial
  work in any `kyriakon-*` repo.
- `docs/CONTEXT.md` — the shared glossary (shell-less user, pricing tiers, dogfooding, audit us).

Repo-local terms and decisions live in the individual `kyriakon-*` repos, not here. If you're
working in a sibling repo, its own `AGENTS.md` points back here for orientation.

## Build & test

No build step — markdown only.

## Code style

- **No secrets, ever, in any tracked file.** This repo publishes the project's decision record; a
  leaked value here is a leaked fact about the platform, not a dev-env token.
- Cross-cutting ADRs live in `docs/decisions/`; shared vocabulary in `docs/CONTEXT.md`. Don't
create repo-local files here — those belong in the relevant `kyriakon-*` repo.

## Never

- Never edit `docs/decisions/*.md` (ADRs) as a side effect of an unrelated task — these record
  decisions, not implementation notes, and changes need explicit human review.
- Never write real secrets, key material, or user data into any tracked file.

## Agent skills

### Issue tracker

Issues live in GitHub Issues, driven via the `gh` CLI. See `docs/agents/issue-tracker.md`.

### Triage labels

Five canonical triage labels, strings equal to their names: `needs-triage`, `needs-info`,
`ready-for-agent`, `ready-for-human`, `wontfix`. See `docs/agents/triage-labels.md`.

### Domain docs

Single-context: shared vocabulary in `docs/CONTEXT.md`, cross-cutting ADRs in `docs/decisions/`.
See `docs/agents/domain.md`.

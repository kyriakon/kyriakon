# kyriakon

Cross-cutting documentation for the kyriakon.net project — a low-extraction hosting
platform for the Orthodox Christian community — and its related repos:

- [`kyriakon-infra`](https://github.com/kyriakon/kyriakon-infra) — Terraform + OpenBSD
  configuration for the platform itself
- `kyriakon-site` — the static kyriakon.net marketing/landing site
- `kyriakon-onboard` — the SSH TUI signup/login service and Stripe-webhook receiver
  (see the project proposal, §5.9) — the platform's first bespoke, non-base-system,
  internet-facing service, so it carries stricter agent guardrails than the others
- [Kleio](https://github.com/kyriakon/kleio) — a related but independently-scoped
  `pass`-compatible password manager; kyriakon.net is a first-class provider in
  Kleio's "add a remote" flow (project proposal, §3), with `kyriakon-onboard`'s TUI
  embedded directly in Kleio's UI

This repo holds the one source of truth for decisions and vocabulary that cut across
more than one of the above — not implementation. Repo-local decisions (e.g. the
OpenBSD-install-automation ADR) stay in the repo they belong to.

## Contents

- `docs/decisions/` — cross-cutting ADRs, starting with the founding project proposal
- `docs/CONTEXT.md` — shared vocabulary used consistently across all `kyriakon-*` repos

## For agents

If you're working in any `kyriakon-*` repo cloned as a sibling of this one, read
`docs/decisions/kyriakon-net-project-proposal.md` and `docs/CONTEXT.md` here before
non-trivial work — each repo's own `AGENTS.md` points back here for orientation rather
than duplicating this content locally.

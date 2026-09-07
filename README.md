# kyriakon

The source of truth for decisions and shared vocabulary across the kyriakon.net project -
secure email, static web hosting, and `pass` repositories. Principled hosting, run by
Orthodox Christians.

This repo holds cross-cutting documentation, not implementation. Each repo below is the
authority on its own code; this one records the *why* and the shared language so they stay
consistent.

## Projects

- [`kyriakon-infra`](https://github.com/kyriakon/kyriakon-infra) - Terraform + OpenBSD
  configuration for the platform itself
- `kyriakon-site` - the static kyriakon.net marketing/landing site
- `kyriakon-onboard` - the SSH TUI signup/login service and Stripe-webhook receiver
  (project proposal, §5.9); the first bespoke, internet-facing service, so it carries
  stricter agent guardrails than the others
- [Kleio](https://github.com/kyriakon/kleio) - the pass-compatible password manager we
  recommend for communal security; independently scoped, but kyriakon.net is a
  first-class provider in Kleio's "add a remote" flow (project proposal, §3)

## Contents

- `docs/decisions/` - cross-cutting ADRs, starting with the founding
  [project proposal](docs/decisions/kyriakon-net-project-proposal.md)
- `docs/CONTEXT.md` - shared vocabulary used consistently across all `kyriakon-*` repos
- `docs/agents/` - agent-facing guidance (triage labels, issue tracker, domain docs)

## For agents

If you're working in any `kyriakon-*` repo cloned as a sibling of this one, read
`docs/decisions/kyriakon-net-project-proposal.md` and `docs/CONTEXT.md` here before
non-trivial work - each repo's own `AGENTS.md` points back here for orientation rather
than duplicating this content locally.

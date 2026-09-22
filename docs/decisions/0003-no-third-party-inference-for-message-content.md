# 0003 Message and application content is never sent to a third-party inference service

Status: accepted
Date: 2026-09-22

## Context

The triage described in ADR 0002 needs a model, and the cheapest and most capable options are
hosted: a typed-decision API, or a frontier model behind an API key. Hosting inference remotely
would put plaintext message content and applicant data in a third party's hands, which is the
same exposure as a classifier on the mail box, relocated rather than removed.

Locality was considered sufficient at first, on the grounds that the operator host fixes
zero-access where the mail box does not. That reasoning covers the platform reading mail. It says
nothing about anyone else reading it.

## Decision

Message content, application data, and anything derived from them are never sent to a
third-party inference service, for any purpose.

The scope is stated deliberately. It covers message content and application text, and the
artefacts derived from them, including scores and logs. It does not cover envelope metadata, which
the mail box already handles deterministically under ADR 0002.

## Consequences

Hosted inference is ruled out for these two jobs permanently rather than deprioritised, so
Jev-class APIs and cloud models are unavailable to this component even where they would be cheaper
or more accurate.

One consequence is easy to miss. If reply drafting or thread summarisation is ever added, it needs
a local model too, because drafting reads the same content. That, and not the triage model itself,
is what would make a larger local model a hardware requirement.

Any future exception has to change this record rather than a configuration value, which is the
point of writing it down.

## References

- ADR 0002, ADR 0004
- Working notes: `kyriakon-infra/docs/planning/research/zero-access-and-local-decision-models.md`

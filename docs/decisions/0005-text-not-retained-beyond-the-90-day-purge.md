# 0005 Text is not retained beyond the 90-day application purge

Status: accepted
Date: 2026-09-22

## Context

Proposal section 5.9.1 purges rejected or abandoned application data after 90 days. The triage
model needs labelled examples to fit its per-question-type temperature calibration, and the
default implementation would keep every text it classified, indefinitely. That would quietly
create a corpus of message content on the operator host, outside the retention rule the platform
already applies to applications.

Ordinary operation needs the decision rather than the text. Filing a message requires a folder and
a confidence; an audit trail requires the same.

## Decision

Only decisions, scores and outcome labels are retained. Message and application text is purged on
the 90-day window that section 5.9.1 already applies, and that window also governs anything resting
on the operator host's classifier or model runtime.

Fitted calibration values are scalars and persist beyond the window.

## Consequences

Calibration is fitted within the window. Once text is purged it cannot be refitted against, so a
change to a question set cannot be validated against older mail and the measurement starts again
from whatever has arrived since.

The operator host holds no long-term corpus of message content, which bounds what the machine is
worth as a target.

This is the clause most likely to be reversed by accident, because retaining text is the default
behaviour of nearly every tool that would otherwise satisfy this design.

## References

- ADR 0002, ADR 0003
- Proposal section 5.9.1, the 90-day purge of application data
- Working notes: `kyriakon-infra/docs/planning/research/zero-access-and-local-decision-models.md`

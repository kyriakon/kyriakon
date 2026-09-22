# 0004 Open-weight model run on hardware the operator controls

Status: accepted
Date: 2026-09-22

## Context

ADR 0003 fixes where the data may go. It leaves a separate question open: whether the model
itself can be inspected. The two are often conflated and are independent. A closed-weight model
can run locally on hardware the operator owns, satisfying the no-third-party rule while remaining
unauditable. A self-hosted endpoint inside a third party's cloud satisfies auditability while
still receiving the content.

The project's posture answers the second question by publishing rather than asserting, which is
what "audit us" means for infrastructure configuration.

## Decision

The model used for triage is open-weight and runs on hardware the operator controls.

The decision binds properties rather than a particular model: typed decisions with calibrated
confidence, open weights, and execution on the operator host. Which model satisfies that is
settled when the build starts, and is expected to change as the class improves.

`convaiinnovations/laya` is the current candidate and nothing more than that, named so the
reasoning stays checkable: Apache-2.0, a typed-decision model returning `choice`, `score` and
`noul` answers with calibrated probabilities, served by either an Apple Silicon MLX runtime or the
upstream torch package.

## Consequences

The specific model is deliberately not part of the decision. This class is moving quickly and its
quality will improve over the months before the component is built, so pinning a model today would
either be silently ignored later or, worse, be treated as binding by someone who read the name and
not the property. What a replacement has to satisfy is the properties above plus the ceilings
recorded in the working notes: confidence fitted per question type before any threshold is
trusted, and honest behaviour on scripts the model was not trained for.

The two properties are recorded separately because either can be violated alone. A closed model run
locally fails this decision and satisfies ADR 0003. A hosted open-weight endpoint satisfies this
decision and fails ADR 0003.

Inspectable weights, calibration procedure and benchmark harnesses make the claim that content is
processed locally verifiable by a reader rather than promised, which is the standard the rest of
the platform's configuration is held to.

The component is not a security control. It sorts; it never sends, deletes, or approves anything.
Publishing its prompts and criteria therefore costs at most filter evasion, which is why a public
repository is the right default for it.

## References

- ADR 0002, ADR 0003
- `kyriakon-infra` `docs/threat-model.md`, for the published-configuration posture
- Working notes, including the model's measured ceilings:
  `kyriakon-infra/docs/planning/research/zero-access-and-local-decision-models.md`

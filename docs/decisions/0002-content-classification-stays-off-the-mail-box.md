# 0002 Content classification stays off the mail box

Status: accepted
Date: 2026-09-22

## Context

Mail is zero-access: the Maildir holds PGP/MIME ciphertext the platform holds no key for,
protecting subject, body and headers. `kyriakon-infra/docs/threat-model.md` enumerates what
follows, including that abuse monitoring runs on metadata only and that no server-side body or
header search is possible.

Automatically triaging mail and account applications would remove real manual work, and the
obvious host is the mail box, which already holds the messages. Running a classifier there would
also keep content away from third parties, so the question looked like a deployment detail
rather than a boundary.

## Decision

Classification of message content runs on the operator host, never on the mail box.

The mail box keeps routing on envelope metadata alone, that is the recipient address via aliases
and plus-addressing. Zero-access protects content, subject, body and headers, and explicitly does
not protect the envelope, so envelope routing is the only server-side sorting the promise allows.

## Consequences

A content classifier on the box would have falsified four published claims: the threat model's
"never holds a private key or anything that can derive one", the AUP's "the platform cannot read
it", the glossary's zero-access definition that a disclosure order "yields ciphertext and no
key", and the recovery phrase's "Never held by the platform". It would also have converted the
threat model's documented honest ceiling, a compelled admin modifying the delivery pipeline,
into the guaranteed steady state.

Locality on the box is not sufficient, and that is the non-obvious part: it fixes disclosure to a
third party but not zero-access, because the platform still reads every message and would also
hold the key.

Boundary condition. Anything serving a second user's mailbox is the platform again. This decision
covers one operator reading his own mail, and does not license a per-user service.

## References

- `kyriakon-infra` `docs/threat-model.md`, "Zero-access mail" and its Consequences section
- `kyriakon-infra` `docs/aup.md`, "Zero-access boundary"
- `docs/CONTEXT.md`, "Zero-access mail" and "Recovery phrase"
- Proposal sections 5.1 and 5.6
- Working notes: `kyriakon-infra/docs/planning/research/zero-access-and-local-decision-models.md`

# 0009 Payment records are keyed by a token, not by the account

Status: accepted, with one question outstanding for an accountant
Date: 2026-09-30

## Context

ADR 0008 refuses a customer list held by a payment processor and puts the payment lifecycle on the
platform's own machine, which makes the platform the system of record for who has paid and until
when. HMRC requires a sole trader to keep financial records for five to six years, which would make
a payment history keyed to the account the longest-lived personal data the platform holds, outliving
the account itself and everything ADR 0005 bounds to 90 days.

The obvious shape, a ledger run alongside the accounts and keyed by username, is the one to justify
rather than assume, because the account already names a person: an OpenBSD account, a mail address
and a published key. Six years of payment history attached to it is a record of who bought what for
longer than any other record on the box.

## Decision

Two records, with different keys and different lifetimes.

The **payment state** lives with the account: the paid-until date, the rail a payment arrived on, and
a reference to the token issued at approval. It is what decides access, and it dies with the account.

The **financial ledger** satisfies HMRC and is keyed by that token. It holds the amount, the date
received, the rail and the resulting paid-until date. It holds no username, no contact address and no
name. The join between token and account exists only for as long as the account does, so deleting an
account leaves a dated payment record that no longer resolves to a person.

The platform collects no legal name and no postal address from a member, at signup or at approval:
a username, a contact email, a mail public key, an optional SSH key, an optional recovery address,
and the payment facts above. The vet is done from the contact address and an optional free-text
field.

## Consequences

A financial record survives deletion without identifying anyone, which is the point of the keying,
and is also the part that needs an accountant to confirm: nothing published states whether HMRC
accepts a ledger that becomes permanently unlinkable once the account is deleted, while remaining
reconcilable throughout the account's life. Until that is answered this decision carries a caveat
rather than a gap.

The ledger cannot answer "who paid this" after deletion. A dispute from a deleted member is
therefore answered from the card processor where the rail was card, and cannot be answered at all
for a prepaid rail, which is a Ceiling this decision accepts rather than hides.

A defect in the join produces a wrong paid-until date, so the join is the one place in the payment
path that warrants a test of its own, and it is the reason the token reference is written with the
account rather than derived.

Hashing identifiers for longer retention stays rejected, consistent with the log minimisation
already recorded: a retained hash is a pseudonymous identifier rather than data minimisation. A
token is the same trade made deliberately, and here it buys unlinkability after deletion, which the
hash did not.

## References

- ADR 0005, text is not retained beyond the 90-day application purge
- ADR 0008, the refusal list, including no customer list held by a payment processor
- `kyriakon-infra` issue #152, the agreement set and retention schedule this decision came from
- `kyriakon-infra` issue #114, the prepaid rail design, whose manual credit is keyed on a token
- `kyriakon-infra` `docs/planning/research/sole-trader-obligations.md`, the retention schedule and
  the questions it flags for an accountant

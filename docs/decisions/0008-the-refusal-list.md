# 0008 The refusal list

Status: accepted
Date: 2026-09-22

## Context

A statement of what the platform will not do is the only assurance here that a reader can check without trusting the operator. Reading the published configuration shows that a component does not exist, which is a stronger check than any claim about how something behaves.

There are two kinds of refusal and they are not interchangeable. A choice is something the platform could build and does not, so a reader can search the configuration and find the absence. A limit is something the platform cannot do, and stating it plainly is the honest form of a ceiling. Stating a choice as a physical impossibility invites the reply that it is merely unbuilt, and stating a limit as a choice invites the reply that it was decided against the user's interest.

## Decision

The refusal list is the spine of the threat model, and every refusal belongs on one side or the other.

Refusals by choice, each verifiable by absence in the published configuration:

- No content classification or scanning on the mail box (ADR 0002).
- No third-party inference for message content (ADR 0003).
- No key the platform can use to read mail. No escrow, no account recovery that restores message content, no server-side decryption of message bodies.
- No retention of message or application text beyond the 90-day window (ADR 0005).
- No interactive shell for a standard-tier user.
- No rejection of a signup on the shape of a key. Key strength is warned about at signup and the minimum is recorded in the acceptable use policy, because the platform cannot verify what a client generated, and a heuristic rejection would turn a guarantee into a support ticket.
- No warrant canary. It is coercible and it needs standing signing infrastructure, where a static statement of capability is checkable against configuration.
- No third-party analytics or scripts on the static and Gemini sites.
- No read receipts, open tracking, or delivery telemetry beyond what SMTP requires.
- No customer list held by a payment processor. Lifecycle state lives on the platform's own machine, which makes a processor one of several ways to extend a date rather than the system of record.

Refusals by limit, each stated as a limit in the threat model:

- A global passive adversary is not defended against. The platform's link and the correspondent's are both observable.
- A correspondent's stored copy is not protected.
- Ciphertext archived today is not guaranteed to stay unreadable. The property is about the present, and a key obtained later or a broken KEM reads history.
- What key a client generated, and whether it is strong, cannot be verified.
- Verifiable boot integrity cannot be provided on a virtual machine. The firmware and the hypervisor are not the platform's to attest.
- Hardware trust below the management engine cannot be provided. No current x86 host provides it.
- Mail already delivered cannot be erased.
- Availability under provider coercion cannot be guaranteed (ADR 0007).
- Suppression cannot be answered with cryptography, only with jurisdiction and diversity.
- Physical coercion of the operator is out of scope, for the reading adversary and for the suppression adversary alike.

## Consequences

Each item is either a configuration a reader can search or a sentence in the threat model. An item with neither does not belong, which is the same rule that governs the prioritised defence list, so one standard checks both documents.

The public rendering lives in `kyriakon-infra`, referenced from the acceptable use policy, and expands each item with how a reader verifies it. This ADR is the binding decision behind that rendering. The two are kept in step by review rather than by generating one from the other, because the public document carries verification steps that are specific to this repository's configuration.

The refusals constrain work as much as they describe it. A request that would add server-side scanning, a processor-held customer list, or a shell for a standard-tier user is refused by this document rather than by taste.

## References

- ADR 0002, ADR 0003, ADR 0005, ADR 0006, ADR 0007
- `kyriakon-infra/docs/threat-model.md`
- `kyriakon-infra` issue #70 and its triage comment

# kyriakon — shared vocabulary

Cross-cutting terms used consistently across `kyriakon-infra`, `kyriakon-site`, and any
future `kyriakon-*` repo. Repo-local terms (implementation-specific, not meaningful
outside one repo) live in that repo's own `CONTEXT.md` instead — e.g.
`kyriakon-infra/CONTEXT.md` for infra-only vocabulary.

## Language

**Shell-less user**:
The standard account tier across the whole platform. No interactive shell. Mail via
IMAP/SMTP, static/gemini site hosting via `sftp` upload, `pass` git repos via
`git-shell`. This is the platform's core safety property — it's what keeps support
burden, resource contention, and moderation load bounded without needing pubnix-style
shell moderation tooling.
_Avoid_: shell user, pubnix user

**Individual tier**:
The MVP pricing tier — flat £20/yr, `user@kyriakon.net` address, shared infrastructure,
generous default quotas.
_Avoid_: standard tier, base tier

**Own-domain tier**:
A pricing tier at £40/yr for parishes, monasteries and small businesses wanting
`secretary@theirparish.org` on their own domain — same shared infrastructure and
isolated mailboxes as the individual tier, up to ten addresses, sold from the release with
its provisioning automated before it (`kyriakon-infra` #188, #190). Does not include
kyriakon.net hosting the parish's DNS zone — see the project proposal, §6.13.
_Avoid_: custom domain tier, parish tier

**Managed instance tier**:
A pricing tier at £150/yr, listed on the site with its price and ordered by hand rather
than built for the early release: a fully separate VPS rather than shared infrastructure
(`kyriakon-infra` #188).
_Avoid_: dedicated tier, enterprise tier

**Dogfooding**:
Specifically: running Oliver's own email on the platform (Phase 1 of the phased plan in
the project proposal) before any other user's mail touches it. Not a generic synonym
for "testing" — dogfooding refers to this specific first-real-mailbox step.
_Avoid_: testing, trial run

**Audit us**:
The platform's core trust posture: all non-secret infrastructure configuration is
published publicly on GitHub rather than asserted as trustworthy. Shorthand used
throughout the proposal and docs for "verifiable, not just claimed."
_Avoid_: transparency (too generic — this term refers specifically to the
publish-the-config practice, not transparency in general)

**Zero-access mail**:
Mail whose stored form the platform cannot decrypt: it holds only the user's public key
and encrypts on ingress, so a disclosure order yields ciphertext and no key. Protects
message content (subject, body, headers), not correspondence metadata — the stronger
property than `encrypted at rest`, where an admin can still decrypt.
_Avoid_: zero-knowledge mail (overstates — the server still sees the envelope and
in-flight plaintext)

**Recovery phrase**:
The user-held, passphrase-protected offline backup of a mail private key — the only path
back from a lost device or key. Never held by the platform; losing both key and phrase is
permanent mail loss.
_Avoid_: password reset, reset phrase

**Operator**:
The person who runs the platform: provisioning, approving applications, and answering mail
sent to the addresses the domain publishes. Today this is Oliver alone.
_Avoid_: admin (that is the `admin@` address and its folder, not the role), owner, sysadmin,
and the bare word for a third party that operates infrastructure — qualify those, as
`threat-model.md` already does with "registry operator"

**Operator host**:
The always-on machine the operator runs for his own work, distinct from the mail box. It
holds his own mail private key, which makes it the only machine where message content is
processed. See ADR 0002.
_Avoid_: client (already means the PGP-capable mail application), personal machine, laptop

**Custodian**:
A body, not an individual, that holds a copy of the platform's encrypted repository and a
snapshot, and never a key and never live mail. Chosen for continuity and for a proven ability to
act, so an archdiocese or a monastery with a stable connection rather than a parish council.
_Avoid_: data escrow, backup host, mirror, federation

**Secondary site**:
The machine a restored copy runs on after the primary is lost, created from the snapshot and the
custodian's copy rather than standing. Distinct from the primary, which is the mail box, and from
the offline SSD copy, which sits with the operator.
_Avoid_: failover, standby, replica, node, federation

**Triage model**:
The local model on the operator host that sorts inbound mail and account applications into
folders and priority. It sorts only: it never sends, deletes, or approves anything.
_Avoid_: classifier, decision model, System One model, Jev

**Reading adversary**:
An authority that wants the content of mail: the wiretap, the disclosure order, the compelled
operator. Defended against cryptographically, by the platform holding no ability to decrypt, so it
is the one threat class where the defence is verifiable by inspection.
_Avoid_: state actor (ambiguous, see "suppression adversary"), government threat actor, nation-level
threat actor

**Suppression adversary**:
An authority that wants the platform gone, unreachable, or its operators silenced, rather than
wanting to read a particular mailbox. It attacks choke points rather than content, so cryptography
does not answer it and jurisdiction diversity is what does. Physical coercion of the operator stays
out of scope for both classes.
_Avoid_: state actor (ambiguous, see "reading adversary"), surveillance (that is the reading case),
censorship (names the mechanism, not the actor)

**Prepaid rail**:
Any payment that does not pass through the card processor: cash by post, cash in hand, or a Monero
subaddress. Credited by the operator, tagged with the rail it arrived on, so the processor is one of
several ways to extend a date rather than the system of record (ADR 0008).
_Avoid_: offline payment, manual payment (names the mechanism, not the rail), crypto (one of the three)

**Paid-until date**:
The authoritative expiry of a member's access, held on the platform's own machine. A successful
payment extends it; a processor event only reports that a card payment succeeded, and a subscription
in arrears never shortens a date the member has already paid for.
_Avoid_: renewal date, expiry date (ambiguous — both the member's access and the processor's
subscription have one)

**Lapsed**:
The account state after a payment deadline passes without renewal. Mail still arrives and stays
readable, and everything already published stays up, while sending, uploading and git pushes stop.
A bounded grace window follows, and it ends in deletion, so the state is not an indefinite free tier.
_Avoid_: read-only (the proposal's older name, which promised less access than the state gives),
expired, suspended

**Suspended**:
The account state during enforcement under the acceptable use policy. Outbound mail stops at once and
the published site and capsule go dark, while inbound mail keeps being accepted and the member can
still reach the account page to see the reason and respond.
_Avoid_: locked, banned, deactivated

**Closing**:
The seven-day window between a member requesting deletion, or being refunded, and the deletion
running. The account works normally throughout, delivery continues, and the closure can be cancelled
from the account page.
_Avoid_: pending deletion, cancellation (that term belongs to the consumer's statutory right)

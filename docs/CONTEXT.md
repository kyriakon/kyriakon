# kyriakon — shared vocabulary

Cross-cutting terms used consistently across `kyriakon-infra`, `kyriakon-site`, and any
future `kyriakon-*` repo. Repo-local terms (implementation-specific, not meaningful
outside one repo) live in that repo's own `CONTEXT.md` instead — e.g.
`kyriakon-infra/CONTEXT.md` for infra-only vocabulary.

## Language

**Shell-less user**:
The standard account tier across the whole platform. No interactive shell. Mail via
IMAP/SMTP, static/Gemini site hosting via `sftp` upload, `pass` git repos via
`git-shell`. This is the platform's core safety property — it's what keeps support
burden, resource contention, and moderation load bounded without needing pubnix-style
shell moderation tooling.
_Avoid_: shell user, pubnix user

**Individual tier**:
The MVP pricing tier — flat £20/yr, `user@kyriakon.net` address, shared infrastructure,
generous default quotas.
_Avoid_: standard tier, base tier

**Own-domain tier**:
Deferred-past-MVP pricing tier (~£50/yr) for parishes/businesses wanting
`secretary@theirparish.org` on their own domain — same shared infrastructure and
isolated mailboxes as the individual tier, just domain-attached. Does not include
kyriakon.net hosting the parish's DNS zone — see the project proposal, §6.13.
_Avoid_: custom domain tier, parish tier

**Managed instance tier**:
Deferred/premium pricing tier (~£150+/yr) offering a fully separate VPS rather than
shared infrastructure. Not part of early build.
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

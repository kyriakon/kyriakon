# Kyriakon.net — Project Proposal

**A principled, low-extraction hosting platform for the Orthodox Christian community — email, static/Gemini hosting, and `pass` git repositories on OpenBSD**

---

## 1. Overview

Most cheap shared hosting either over-promises (unlimited everything, funded by upsells and data-adjacent business models) or under-delivers on trust (opaque ops, no way to verify what the provider can actually see). Old-school pubnix communities (SDF, tilde.club) got trust right but scoped themselves around shell access, which brings a support and moderation burden kyriakon.net deliberately wants to avoid at this stage.

Kyriakon.net's core bet is to combine the pubnix community model with a **deliberately narrow, no-shell surface area**: email, static/Gemini site hosting via FTP upload, and `pass`-compatible git repositories via `git-shell`. Every user-facing service maps to a single well-understood OpenBSD base-system daemon, and the entire non-secret configuration is published on GitHub — **"trust us" replaced with "audit us."**

**Non-negotiable constraint**: the platform never holds plaintext of a `pass` store. GPG-encrypted repos mean the server is a dumb git remote; this is a genuine trust property, not a marketing claim, and it must remain true through every future feature addition.

Kyriakon.net is the natural hosting-side counterpart to **Kleio** (a separate, standalone `pass`-compatible password manager project): kyriakon.net offers itself as one *suggested* SSH git remote in Kleio's "add a remote" flow, but has no other coupling — it works as a plain git-over-SSH remote for anyone, Kleio user or not.

## 2. Design Principles

- **Narrow surface, not shell access.** Every capability maps to one base-system OpenBSD daemon operating in a restricted mode (`httpd`, `ftpd` chroot, `git-shell`, `gmid`). No general-purpose shell for standard accounts. This is the single decision that keeps support burden, resource contention, and moderation load bounded as the user base grows.
- **Audit us, don't trust us.** All non-secret infrastructure configuration — `pf.conf`, `httpd.conf`, `smtpd.conf`, Dovecot config, `sshd_config`, `gmid.conf`, provisioning scripts — is public on GitHub from day one. A `threat-model.md` states plainly what the platform protects against and what it doesn't, and what the admin can and cannot see.
- **Accessible, not extractive.** Pricing is set at real break-even plus a small buffer, not tiered to maximise revenue. £30–150/yr tiers were explicitly considered and rejected as inconsistent with the mission.
- **Genuinely open to non-Orthodox users**, while being built by and primarily for the Orthodox community, distributed through parish/diocese trust networks rather than paid acquisition.
- **Filesystem-native storage over databases wherever the workload allows it** — Maildir over a mail database, plain git repos over an application-layer store — because it's simpler to reason about, trivially backed up via `rsync`, and has no separate engine to operate or lose data in.
- **Mail at rest is encrypted.** Maildirs are never stored plaintext on disk — the "audit us" trust posture would be hollow if a disk walking out the door exposed every message. Resolved: both full-disk `softraid` encryption *and* per-message Dovecot `mail_crypt` (§5.1) — FDE for a stolen disk, `mail_crypt` so the Maildir itself stays non-plaintext even on the backup copy.

## 3. Naming & Identity

- **Name / domain**: kyriakon.net (Κυριακόν — "of the Lord," the root of "church" in the Germanic-language line: Kirk, Kirche). `kyriakon.org` is already registered by an unrelated party — `.net` is the sole domain going forward, so anywhere the platform needs to reference itself in copy, docs, or config, `.net` is canonical and no `.org` fallback/redirect logic is needed.
- **Relationship to Kleio**: kyriakon.net is a standalone hosting platform — Kleio remains a separate, independently-scoped project, and neither should silently grow a hard *technical* dependency on the other's internals. That said, this is a deliberately closer integration than "just another remote": kyriakon.net is a **first-class provider inside Kleio's "add a remote" flow**, with its SSH-based signup/login TUI (§5.9) embedded directly in Kleio's UI, not merely linked out to. Other remotes stay generic SSH-key-only in Kleio, with no equivalent embedded flow. This is a conscious choice to make the combination easy for non-technical users specifically, not an accidental coupling — but it should stay a one-way UX integration (Kleio knows how to talk to kyriakon.net's onboarding protocol) rather than either project's core data model or account logic depending on the other's internals.
- **kyriakon.net promotes Kleio, not the other way round.** The public site should recommend Kleio as *the* password manager to pair with a `pass` git repo hosted here, and advertise it as a feature of the platform — see §7.
- **Repo names**: `kyriakon-infra` (infrastructure, §5.6) and `kyriakon-onboard` (the onboarding service, §5.9 — planned as its own repo) are separate public repos; the marketing/landing site lives in `kyriakon-site`. All hold everything except secrets.

## 4. Tech Stack

| Layer | Choice | Notes |
|---|---|---|
| Host | Hetzner VPS, OpenBSD 7.x | no native OpenBSD image on Hetzner — manual install or snapshot, see §6.1 |
| Mail transport | OpenSMTPD | ships in OpenBSD base |
| Mail retrieval | Dovecot (IMAP) | package, not base system — patch cadence differs from base, see §6.10 |
| Mail storage | Maildir | filesystem-native, `rsync`-backed, no DB engine; encrypted at rest (§2) |
| Anti-spam | `spamd` (base) + rspamd | greylisting + content classification into a Junk folder, see §5.1 |
| Static sites | OpenBSD `httpd` | virtual host per user |
| TLS | `acme-client` (base) | automatic Let's Encrypt |
| Site upload | `ftpd` (base) | chrooted, isolated per-user directories |
| Gemini protocol | `gmid` | C, OpenBSD-native — chosen over Agate/Rust for ecosystem-ethos fit, not performance |
| Git / `pass` hosting | `git-shell` | restricted shell, git-only, SSH key auth only |
| Firewall | `pf` (base) | packet filtering, rate limiting |
| TLS termination (if needed) | `relayd` (base) | |
| Backups | encrypted `rsync`, nightly, separate Hetzner storage box | Maildir + git repos + web roots |
| IaC | Terraform, `hcloud` provider | public repo, provisions the VPS |
| DNS | `nsd` (hidden primary, ports) + Hurricane Electric (free secondary) + Cloudflare (CNAME/Partial setup for proxied hostnames) | see §5.7, §6.13 — Cloudflare's own Secondary DNS product is Enterprise-only |
| Abuse monitoring | cron scripts + external notification channel | see §5.8 |
| Onboarding & account portal | bespoke Rust service (small HTTPS listener behind `relayd` for signup + Stripe webhook in MVP; `russh` for the SSH TUI as a fast-follow) | see §5.9 — the platform's first custom, internet-facing, auth-handling service; not an OpenBSD base daemon |
| CGI (phase 2) | `slowcgi` (base) | deferred — see §7 |

## 5. Architecture

### 5.1 Mail

OpenSMTPD + Dovecot + Maildir, with SPF/DKIM/DMARC configured from day one rather than bolted on later. `spamd` greylisting at the base-system level, plus **rspamd + sieve** for content classification into a Junk folder — resolved: raw spam volume is too high for greylisting alone to be an acceptable user experience. Deliverability is treated as an ongoing operational discipline, not a one-time setup step — see §6.2.

Mail storage is encrypted at rest (§2): Maildirs are never stored plaintext on disk. Resolved: **both** — full-disk `softraid` crypto (protects a stolen/pulled disk, near-zero cost on AES-NI hardware) and per-message Dovecot `mail_crypt` (keeps the Maildir itself non-plaintext at rest and on the backup copy). Accepted costs: `mail_crypt` degrades server-side IMAP SEARCH and adds per-user keypair management.

**Authentication is via real OpenBSD system accounts, not a separate virtual-user database** (see §5.9): each user is a genuine `useradd`-created account with shell forced to `/sbin/nologin`, home directory holding their Maildir, and a real system password. Dovecot and OpenSMTPD authenticate against these accounts via PAM — the nologin shell blocks any interactive session (console, `su`, or a hypothetical SSH password auth) without affecting PAM authentication at all, since PAM doesn't consult the shell field. This is the traditional Unix mail-hosting pattern, and fits §2's filesystem-native-over-database principle even better than a Dovecot-native passdb would: one real account per person, one home directory holding Maildir *and* their git repos (§5.3) *and* their site content (§5.2), one password.

### 5.2 Static & Gemini hosting

`httpd` serves a virtual host per user at `username.kyriakon.net` — the username chosen at signup (§5.9) is also the public subdomain, so this needs a single wildcard DNS record (`*.kyriakon.net`) rather than a DNS write per signup, which fits the "audit us" git-tracked zone in §5.7 cleanly (no runtime zone mutation on account creation). `acme-client` handles TLS for the platform's own fixed hostnames automatically, but **a wildcard cert needs a different route**: `acme-client` implements HTTP-01 only, not DNS-01, and always has (confirmed against the current OpenBSD man page — not a version gap, a permanent scope decision) — a wildcard cert isn't obtainable through it at all. `*.kyriakon.net` is routed through Cloudflare's proxy (extending the same CNAME/Partial pattern already scoped in §5.7 for other hostnames), and Cloudflare's Universal SSL covers the wildcard. Users upload via `ftpd`, chrooted and isolated per-user — no shell needed to publish a static site. The same upload path serves `.gmi` files for `gmid`, so Gemini capsule hosting is close to free once static hosting works: one upload mechanism, two serving daemons — but see §6.9: the exact directory layout (shared root vs. sibling `www/`/`gemini/` directories per user) needs deciding before the first real upload, not worked out after users have content in place.

### 5.3 Git / `pass` hosting

`git-shell` gives per-user, SSH-key-only, git-operations-only access under `/home/user/repos` — no interactive shell, so this doesn't reopen the shell-access surface the platform is otherwise avoiding. Because `pass` stores are GPG-encrypted client-side, the server never sees plaintext; this should be stated explicitly and prominently in user-facing materials, not just in the threat model, since it's the platform's strongest trust claim for the audience it's built for.

§6.8 flagged an open question about `git-shell`'s coarse access control — **now resolved**: since real OS accounts are required anyway for mail auth (§5.1), "one OS user per person" is simply the model, not a separate decision, and it gives `git-shell` clean per-user isolation for free via the account's own home directory. `gitolite` is no longer needed to solve this problem.

### 5.4 Firewall & network

`pf` for packet filtering and rate limiting; `relayd` for TLS termination if a service needs it that `httpd`/`acme-client` doesn't already cover.

### 5.5 Backups

Nightly encrypted `rsync` to a separate Hetzner storage box, covering Maildir, git repos, and web roots. **Key custody (resolved):** the backup-decryption key (and any FDE passphrase) lives in a separate offline copy held by Oliver only for now, off the box — not resident on the primary, since a key stored only on the primary makes the backup undecryptable exactly when the primary dies. Given how unforgivable mail loss is to users, **the restoration procedure must be tested on a real schedule, not assumed to work because the backup job runs green** — this is the single highest-consequence item in the whole platform and should get a disproportionate share of early engineering attention relative to its apparent complexity.

### 5.6 Public infrastructure repo (`kyriakon-infra`)

```
kyriakon-infra/
├── terraform/           (main.tf, variables.tf, outputs.tf)
├── openbsd/etc/          (pf.conf, httpd.conf, smtpd.conf, login.conf, ssh/sshd_config)
├── openbsd/dovecot/
├── openbsd/gmid/
├── openbsd/nsd/          (zone files, nsd.conf — see §5.7)
├── scripts/              (provision.sh, add-user.sh, del-user.sh, backup.sh, abuse-monitor.sh — see §5.8)
└── docs/                 (architecture.md, security.md, threat-model.md)
```

Everything here is public except secrets themselves (DKIM private keys, TLS keys, API tokens, user data). `threat-model.md` states plainly what the platform protects against (passive surveillance, data mining by the platform itself) and what it explicitly does not (a determined state actor, a user's own compromised key) — and what the admin can see (delivery metadata) versus cannot (encrypted `pass`-store content). A `.gitignore` plus secret-scanning pre-commit hooks (see §8, `betterleaks`-style tooling as already used on Kleio) guards against accidental secret commits — this matters more here than on a typical app repo, since a leaked key on this repo is a leaked key to real users' mail and git access, not just to a dev environment.

A sibling repo, **`kyriakon-onboard`**, holds the bespoke onboarding service (§5.9) — including the reserved-name data file (§5.9.3) — rather than this repo.

### 5.7 DNS

Zone files are the source of truth, held in `openbsd/nsd/` in this repo — fitting the "audit us" principle in §2 arguably better than a third-party dashboard would, since the zone itself becomes reviewable in the same PR flow as everything else.

`nsd` runs as a **hidden primary** on the platform's own box: it answers zone-transfer requests (AXFR, NOTIFY-triggered) from secondaries, but is never itself listed in public NS records, so it's not a direct target for DNS-layer probing or DDoS. Two public-facing roles sit in front of it, kept deliberately separate because they solve different problems:

- **Hurricane Electric free secondary DNS** answers ordinary public queries for the zone (MX, SPF/DKIM/DMARC, NS, and any A/AAAA records not proxied through Cloudflare). Free, well-established for exactly this hidden-primary pattern, and — critically — keeps DNS resolution up even if the origin box is briefly down, since HE's anycast network continues serving the last-transferred zone (see §6.11).
- **Cloudflare, via a CNAME (Partial) setup**, covers just the specific hostname(s) that benefit from its proxy — the static site's public hostname, primarily. This is the standard "keep your own DNS provider, only proxy this one subdomain" pattern and doesn't require Cloudflare to be the zone's authoritative provider at all. Cloudflare's actual Secondary DNS product (either direction — Cloudflare as primary or as secondary for a whole zone) is Enterprise-only and not economically proportionate to this project; the CNAME/Partial route gets the useful part (origin-IP hiding, DDoS absorption, edge caching for the proxied hostname) without that cost.

**Protocol limit, not a configuration gap**: mail (SMTP/IMAP) and `git-shell` (SSH) can't be proxied through Cloudflare regardless of DNS setup — MX and any SSH-facing record must resolve directly to the box's real IP for the protocol to work at all. The hidden-primary pattern protects the *DNS answering service* specifically; it doesn't and can't hide the mail or git endpoints themselves. Gemini (`gmid`) is in the same position — no proxy layer exists for the Gemini protocol, proxied or otherwise.

This is scoped as a Phase 4 hardening item (§9), not MVP-blocking — Cloudflare-as-sole-provider (the platform's current setup) is a perfectly reasonable starting point, and the hidden-primary migration can happen once the core services are stable without disrupting anything user-facing.

### 5.8 Automated abuse monitoring & notification

Cron-driven scripts (`scripts/abuse-monitor.sh` and siblings) watch for the signals that actually predict abuse or compromise on a platform this shape, and notify Oliver automatically rather than relying on him noticing:

- Sudden outbound mail-volume spikes from a single mailbox (the classic signature of a compromised account being used to relay spam)
- Repeated authentication failures against SMTP/IMAP/SSH, per-account and per-source-IP
- A user's storage approaching or hitting their `edquota` limit (§6.5)
- New entries in `spamd`'s greylist/blocklist state that weren't there before
- The platform's own sending IP appearing on a reputation blocklist (folds in the ongoing check from §6.2/§6.12 rather than leaving it purely manual)

**Notifications must go over a channel that doesn't depend on the box being healthy or trustworthy** — if the box itself is compromised and relaying spam, its own outbound mail is exactly the thing you can no longer trust to reach Oliver reliably. Resolved: **Healthchecks.io** is the channel, doubling as both the external ping target for health-monitoring (§6.11, layer 1) and the dead-man's-switch alert path — one external dependency that doesn't ride on the platform's own mail.

This is scoped as in-scope for MVP alongside the mail stack itself (§7) — the value is highest before real users exist to generate false-positive noise while the thresholds are still being tuned, not after.

### 5.9 Onboarding & account portal

**Split into two phases, not just for scheduling reasons but because it's a genuinely smaller, sounder MVP this way**: a conventional HTTP signup form + Stripe Checkout ships in MVP v1; the SSH TUI (§5.9.2) — the harder, more novel part — is a fast-follow once the core platform is stable. This removes the need to write a custom SSH server (`russh`) for MVP at all: the "first bespoke, internet-facing, authentication-handling service" concern from §6.14 shrinks to "a small HTTP service behind `relayd`," which is a far more conventional, well-trodden thing to build and secure correctly on a solo timeline.

**One knock-on effect worth being explicit about**: the tightly-embedded Kleio integration described in §3 — the SSH TUI rendered inline inside Kleio's UI — depends on the TUI existing, so it moves with it to the fast-follow phase. For MVP, Kleio's "add kyriakon.net as a remote" is functionally the same as adding any other SSH remote: the user pastes the SSH pubkey they registered during web signup, and Kleio connects to `git-shell` normally. kyriakon.net is still promoted as Kleio's recommended pairing (§3, §7) — that's a content/positioning decision, unaffected by which signup mechanism is live — but the *embedded* experience specifically is fast-follow, not MVP.

#### 5.9.1 MVP: HTTP signup + Stripe Checkout

A small Rust web service (e.g. `axum`), sitting behind `relayd` for TLS termination, at a fixed hostname (this is *not* the wildcard-cert problem in §5.2 — a fixed signup hostname is ordinary `acme-client` HTTP-01 territory, no Cloudflare routing needed for this specific piece).

1. Web form collects: desired username (validated live against the reserved-name list, §5.9.3, so a taken/reserved name is caught before payment, not after), email, a mail password, and — optionally, can be added later from the account page — an SSH public key for `git-shell`.
2. Submitting the form creates a Stripe Checkout Session server-side, carrying a provisioning-session token as the session's correlation reference, and redirects the user to Stripe's hosted Checkout page.
3. User pays entirely on Stripe's own domain — the platform never touches card data, no PCI scope.
4. Stripe fires a webhook at a small endpoint on the same service. **The webhook signature must be verified against Stripe's signing secret and any request that fails verification rejected outright** — an unverified "payment succeeded" endpoint is a free-account-creation vector, not a convenience.
5. On verified payment, the service provisions a real OpenBSD system account (`useradd`, shell forced to `/sbin/nologin`, per §5.1), sets the system password to the one chosen in the form (this doubles as the mail credential, §5.1), and registers the SSH key if one was supplied. The user is redirected to a confirmation page; if the browser tab closes before this redirect, the account is already provisioned regardless (payment confirmation, not the redirect, is what triggers provisioning) — worth stating plainly since it's the kind of edge case that's easy to get backwards.

**Account lifecycle (resolved):** deletion is admin-mediated, not automated — GDPR erasure is a hard obligation (§6.3), but a human performs the final delete, not a script firing blind. On failed renewal the account drops to read-only and the user is emailed; after 40 days without resolution, the abuse-monitor/Healthchecks path (§5.8) notifies Oliver to reach out personally — a parish whose "technical user" has gone AWOL is exactly the case a blind auto-delete would get wrong — then `scripts/del-user.sh` removes the mailbox, git repos, web/Gemini content, and OS account, and cancels the Stripe subscription.

#### 5.9.2 Fast-follow: SSH TUI

Once the core platform is stable, an additional signup/login entry point over SSH, and the mechanism that unlocks the embedded-in-Kleio experience from §3:

- A bespoke Rust service (`russh`), **on its own port, separate from stock `sshd`** — not a layer in front of it. SSH's encryption is negotiated end-to-end between client and whichever process completes the handshake, so there's no way to terminate the protocol in one process and hand the live session to a second, independent `sshd` that never participated in the key exchange. The two workable shapes are (A) one unified `russh` server on port 22 replacing `sshd` entirely, execing `git-shell` as a subprocess for known-key connections and running the TUI in-process otherwise, or (B) `sshd` on 22 untouched — exactly as already scoped for `git-shell` — with this service on its own port entirely. **(B) is the chosen approach**: it keeps the platform's narrow-single-purpose-daemon pattern (§2) intact even for this new component, and means a bug or abuse flood against the new, unproven code can't degrade or crash the process that established `git-shell` users depend on. `pf` rate-limits the new port independently of everything else.
- Cold-start signup: same Stripe Checkout Session + webhook mechanism as §5.9.1, just reached from the TUI instead of a web form — printing the Checkout URL as text and a QR code, so a phone can complete payment while the terminal session stays open, then polling or an explicit "continue" once paid.
- Returning-user login: password + email against the same real system account/PAM mechanism as IMAP (§5.1), or SSH pubkey for technical users who'd rather not type a password.
- Embedding this inside Kleio's "add a remote" flow (§3) becomes possible once it exists — Kleio would need genuine terminal-emulation capability in its UI (a `portable-pty`-plus-terminal-widget, not just an SSH client library), which is its own scope on Kleio's side worth logging there when this is actually scheduled.
- **Optional key backup (idea, not a decision):** since the server never holds plaintext (§5.3), a user who loses their GPG key has no recovery path. `pass` data-loss and recovery is Kleio's subject, not this platform's — but the TUI flow could optionally offer a key backup unlocked by a recovery code. Logged here as an idea to carry into Kleio's scoping, not committed.

#### 5.9.3 Reserved-name list (applies to both phases)

**The username is also the public subdomain** (`username.kyriakon.net`, §5.2) and the OS account name, so validation has to satisfy OS-account constraints *and* screen for names that would be actively harmful if minted, not just malformed ones. Hard requirement, not a nice-to-have: alphanumeric-only, length-capped, and checked against a maintained reserved-name list before it ever reaches `useradd` or a filesystem path. That list needs at least four categories, kept as a single data file in `kyriakon-onboard` (not scattered inline) so it's easy to extend as the project grows:
- **Infrastructure/protocol names** that would collide with a real subdomain or service: `www`, `mail`, `smtp`, `imap`, `mx`, `ns`, `ns1`, `ftp`, `git`, `api`, `cdn`, `static`, `docs`, `status`, `gemini`, `onboard`, `admin`, `root`, `webmaster`, `postmaster`, `abuse`, `support`, and similar.
- **Impersonation/authority-sounding names**: `login`, `signin`, `verify`, `secure`, `official`, `security`, `service`, `team`, and similar generic-authority words — the ones most likely to be combined with a brand name for a phishing-style subdomain (`paypal-verify.kyriakon.net`). **This category is inherently incomplete by design** — a fixed list can't enumerate every brand or combination an attacker might try, so it's a first line of defence, not the whole defence; §5.8's abuse monitoring and a reporting path are the backstop for what the list misses, not a redundant afterthought.
- **Project/brand reserved names**: `kyriakon`, `kleio`, and placeholders for known future ventures (`dev` for software contracting, `press` for Orthodox liturgical publishing) and existing open-source projects (e.g. `cssfingerprint`, `orthodoxcalendar`) — plus, going forward, **the name of every `kyriakon-*` repo or service gets added to this list before it launches**, as a standing rule rather than something remembered case by case.
- **Inappropriate/offensive names**: checked against an established open-source wordlist as a starting point. Worth being honest that exact-match filtering alone is bypassable (leetspeak substitution, separators, Unicode homoglyphs) — normalizing input before comparison closes some of this, but not all of it, and a manual reporting/review path is a proportionate backstop for a small, trust-based platform (§2) rather than trying to build a fully automated classifier for this at MVP scale.

**This is the platform's first custom, internet-facing, authentication-handling service** — everything else leans on decades-hardened OpenBSD base daemons in restricted modes; this is bespoke software the project is fully responsible for hardening itself. Code public, well-tested, per the project's existing "audit us" standard (§2) — but that standard needs to be actually met here, not just asserted, given what this component is trusted with (OS-account provisioning, payment correlation, credential handling).

**This is a substantial scope addition**, comparable in size to the rest of MVP combined — an internet-facing Rust service handling auth, payment correlation, webhook verification, and provisioning isn't a small add-on. Resolved: the HTTP signup + Stripe Checkout slice (§5.9.1) ships in MVP v1; the SSH TUI (§5.9.2) is a fast-follow once mail/static/Gemini/git are stable and dogfooded.

## 6. Known Risks & Open Engineering Questions

### 6.1 OpenBSD install automation — resolved: snapshot-first

Hetzner has no native OpenBSD image, so every install currently starts from manual rescue-mode work. Two paths were considered:

1. **Autoinstall response file** (rejected for now) via Hetzner rescue mode + Terraform `remote-exec` — fully scriptable, but response-file syntax against Hetzner's rescue environment is unproven and would need its own spike.
2. **Snapshot approach (chosen)** — install and configure manually once, snapshot via the Hetzner API, and use the snapshot as the Terraform image source from then on. Front-loads the one-time manual pain instead of trying to eliminate it, which is the more realistic target given the size of this project.

Community "openbsd-hetzner" scripts exist and are worth auditing before writing anything from scratch, but should not be trusted uncritically given this box will hold real user mail and keys.

**Decision: snapshot-first, no autoinstall spike.** The spike is deliberately skipped — the snapshot approach is the realistic path for a solo build, and the autoinstall route can be revisited only if snapshot-based reprovisioning turns out to be painful in practice. **This item blocks everything else** — nothing in §9's phased plan can start until the snapshot-based provisioning path exists, so it's the correct first step.

### 6.2 Mail deliverability is ongoing, not one-time

SPF/DKIM/DMARC, correct PTR records, an IP warm-up period, and avoiding pre-blocked AWS/GCP-style ranges (not a concern on Hetzner, but worth confirming Hetzner's own ranges aren't flagged) all need to be right from day one, then monitored continuously via MXToolbox/mail-tester.com. Resolved: no formal warm-up campaign — organic low-volume usage (dogfood first, then a handful of beta users) *is* the warm-up at this scale. The one cheap gate kept: check the assigned Hetzner IP against Spamhaus and other reputation lists *before* the first send (§6.12), then monitor.

**DMARC policy (resolved):** start `p=none` with reporting, tighten to `p=quarantine` then `p=reject` once SPF/DKIM alignment is proven stable through the dogfood + beta phase. **DKIM key rotation (resolved):** rotate yearly or on-compromise, via a small script, with a published `t=s` revocation path in the runbook — not MVP-blocking.

### 6.3 GDPR / data-controller obligations

Oliver is a data controller even at this scale. Needs a privacy policy and a stated lawful basis before onboarding real users — not onerous work at this size, but it's a hard gate before §7's "closed beta with 3–5 trusted users," not something to defer past it.

### 6.4 Abuse handling

Needs a written acceptable-use policy and, more importantly, genuine willingness to act on it even against known community members — a trust-based, word-of-mouth platform is exactly the kind of environment where enforcement against a known contact is socially hardest and most likely to be skipped if it isn't decided on in advance, before there's a specific person involved. Automated detection (§5.8) removes the "didn't notice" excuse; it doesn't remove the harder social one, which the written policy exists to pre-commit against.

### 6.5 Storage creep

No quotas by default is how a generous-quota platform quietly runs out of disk. Resolved: **5 GB per account**, enforced via `edquota`, across mail + web + git combined — picked as the floor that clears Posteo (2 GB) by 2.5× while leaving ~200-user headroom on a single 1 TB storage box (§5.5). It's a positioning + abuse-defense number, not a cost-recovery one (raw disk is ~3.6p/GB/yr), and it's upgradeable later — reducible only via a migration, so it starts low. `edquota` enforcement lands alongside the mail/hosting features it applies to, not as a later hardening pass.

### 6.6 Becoming critical infrastructure

Once a parish depends on kyriakon.net for its email, "Oliver's side project" and "the parish's mail server" are the same box. Resolved: **Marios (the Kleio co-contributor) holds the second root credential**, no one else for now — recorded in the runbook before the first parish account, not after, along with the backup-restoration procedure and who to contact if Oliver is unreachable. **Admin access model (resolved):** root over SSH is disabled; admins log in with personal keys as unprivileged users and escalate via `doas`, confining the root surface to `doas` rules and giving a per-admin audit trail of who became root.

### 6.7 Shell access, if ever reconsidered

Deliberately out of scope now (§2), but flagged because it's the most likely future feature request from technically-inclined users. If it's ever added: `login.conf`/`ulimit` resource limits and a real moderation plan need to exist *before* the feature ships, not be retrofitted once someone's runaway process is affecting other users.

### 6.8 `git-shell`'s access-control model — resolved

Originally flagged as an open risk (`git-shell` alone doesn't give one-key-to-one-repo isolation without either one OS user per person or a tool like `gitolite`). **Now resolved by §5.1/§5.9's requirement that every user is a real OpenBSD system account** (needed for mail auth regardless): one account per person gives `git-shell` clean isolation via the account's own home directory as a side effect, so this is simply the model rather than a decision still to make. `gitolite` is no longer under consideration.

### 6.9 Static and Gemini directory layout — resolved: sibling `www/` + `gemini/`

Reusing the `ftpd` upload path for both `.html` and `.gmi` content is efficient (§5.2), but `httpd` and `gmid` don't agree on per-vhost directory conventions. **Resolved: sibling `www/` and `gemini/` directories per user**, not one shared root — removes any content-type ambiguity between the two daemons' vhost conventions, and it's decided before the first beta upload so there's nothing to migrate. Cost is two paths in the `ftpd` chroot instead of one.

### 6.10 Package vs. base-system patch cadence

Dovecot ships as an OpenBSD package, not part of base — its security-patch cadence is independent of `syspatch`/base-system updates. The snapshot-based provisioning approach (§6.1) needs to account for this explicitly: reprovisioning a box from an old snapshot should not silently reintroduce a Dovecot version with a since-patched vulnerability. Worth a line in `provision.sh` or the operational runbook stating how package updates are tracked separately from base-system ones. **Base-OS release cadence (resolved):** OpenBSD releases every 6 months, each supported ~1 year — `sysupgrade` runs on a stated cadence (per release or per two releases), captured in the runbook, with Healthchecks.io (§5.8) configured to email Oliver when an upgrade is due or overdue.

### 6.11 Single VPS is a single point of failure for mail delivery, not just uptime

An outage on the one box doesn't just mean downtime — mail sent to it *during* the outage is often not queued and retried by the sending server the way people assume; it can bounce or silently fail upstream, meaning real message loss rather than delay. Resolved: not fully accepted as-is — the secondary MX (layer 3 below) is pulled into Phase 3, before the closed beta, so the message-loss risk is addressed rather than carried through beta.

Rather than one big "make it redundant" decision, this is better treated as layers adopted roughly in order of cost and complexity, each solving a different slice of the problem:

1. **External health monitoring (cheap, MVP-scope)** — an outside vantage point (**Healthchecks.io**, chosen in §5.8 where it doubles as the abuse-alert channel) checking SMTP/IMAP/HTTPS/Gemini reachability and alerting the moment the box goes dark. This has to be genuinely external, for the same reason as §5.8's notification channel: monitoring hosted only on the box itself is exactly the thing that goes silent when the box does. Shortens detection time; doesn't prevent the outage.
2. **Tested backup restore (already in scope, §5.5)** — the existing nightly-`rsync`-plus-tested-restore plan is the actual recovery mechanism once an outage is detected. This is the highest-leverage item on the list and is already MVP-scope; layers 3–4 below are refinements on top of it, not replacements for it.
3. **Secondary MX / backup mail relay (cheap, Phase 3)** — a low-priority MX record pointing at a **second, much smaller OpenBSD box** that queues incoming mail during a primary outage and relays it once the primary is back, directly addressing the message-loss risk described above without needing a fully redundant mail stack. Not a third-party backup-MX service — that would mean a third party transiently queueing real user mail during every outage, against the platform's own-infrastructure posture. **Deliverability plumbing is part of the same step:** the secondary needs its own PTR record and inclusion in the primary's SPF so relayed mail isn't spam-filtered on the way back. Pulled into Phase 3, before the closed beta (§9).
4. **Warm-standby VPS (Phase 4+, cost-gated)** — a second Hetzner instance, built from the same snapshot as §6.1, kept stopped (Hetzner charges primarily for storage on a stopped instance, not compute) and ready to be started and pointed at DNS/MX manually if the primary is lost outright. This is a natural extension of the snapshot-based provisioning already planned, not a separate architecture — but it roughly doubles the storage cost, so it's worth deferring until real user volume justifies it rather than paying for it against an empty platform.
5. **Splitting mail/web/git across separate boxes (explicitly not recommended at this stage)** — would limit blast radius per outage, but directly undercuts the "one modest VPS serves hundreds of low-traffic accounts" economics from the original break-even reasoning, for a benefit (partial rather than total outage) that matters far less than actual redundancy (layers 3–4) at this user count. Worth revisiting only if the platform outgrows a single box on resource grounds, not as a resilience measure in its own right.

DNS resolution itself is handled separately via the hidden-primary + Hurricane Electric pattern in §5.7/§6.13, which already solves the DNS-specific instance of this problem independent of the mail/web/git layers above.

### 6.12 Shared-provider IP reputation

The deliverability plan in §6.2 already covers avoiding pre-blocked AWS/GCP-style ranges, but Hetzner IP space carries its own reputation dynamics as a budget/VPS-heavy provider with its own history of abuse originating from it — worth checking the specific assigned IP against reputation lists (Spamhaus, etc.) *before* committing to it as the mail-sending address, not just monitoring deliverability after the fact once mail is already flowing from it.

### 6.13 DNS architecture: hidden primary + Hurricane Electric + Cloudflare (Phase 4, not MVP)

Earlier framing in this document (before this revision) recommended staying on Cloudflare alone and dismissed self-hosting outright. Worth correcting explicitly: **Cloudflare's own Secondary DNS product — in either direction, Cloudflare as primary or as secondary — is Enterprise-only**, and not proportionate to a £20/yr flat-rate platform. That specific combination isn't available at any budget this project would sensibly pay.

The underlying idea is still worth doing, just via a different public-facing secondary than Cloudflare. §5.7 lays out the resulting architecture: `nsd` as a hidden primary on the platform's own box (zone files git-tracked in `kyriakon-infra`, fitting §2's "audit us" principle), Hurricane Electric's free secondary DNS service answering public queries and pulling zone updates via AXFR/NOTIFY, and Cloudflare kept in play via a CNAME (Partial) setup scoped to just the hostname(s) that benefit from its proxy — no Enterprise plan required for any part of this.

What this buys, concretely: the nameserver itself is never a public target (no direct DDoS/probing surface on the box), zone data has a git-auditable source of truth rather than living solely in a third-party dashboard, DNS resolution survives a box outage independent of mail/web/git (folding into the layered mitigation in §6.11), and Cloudflare's proxy/DDoS/edge-caching benefit is retained for exactly the traffic that can use it.

What it doesn't buy: mail and `git-shell` still resolve directly to the box's real IP regardless of DNS architecture — SMTP and SSH aren't proxyable protocols, so "hiding" the platform's IP via DNS only ever applies to the specific hostnames routed through Cloudflare's proxy, not the platform's mail or git endpoints.

On the second, separate question — **becoming the DNS host for parishes' own domains, once the own-domain tier exists** — the reasoning from the original version of this section still holds: full zone hosting for parish-owned domains would need this same hidden-primary pattern run per parish domain, which is workable in principle (the free-secondary approach removes the "need a second always-on box" objection that made this look unattractive originally) but is still meaningfully past MVP. The own-domain tier as scoped in §7 only requires a parish to add MX/SPF/DKIM/CNAME records at whatever DNS they already use — a documented-steps problem, not an infrastructure one — and should stay that way until there's a specific "we don't want to touch DNS at all" request from a real parish to justify taking on zone hosting for someone else's domain.

**Sequencing**: this whole section is scoped as a Phase 4 hardening item (§9), to be done once the core mail/static/Gemini/git services are stable — Cloudflare-as-sole-provider (today's setup) is a perfectly reasonable place to stay through the MVP and closed beta. Within Phase 4, the DNS migration runs ahead of CGI: it's free, pure config, and lowers the public attack surface with no user-facing risk, whereas CGI carries real sandboxing work and has no demand yet (§9).

### 6.14 Onboarding service's trust surface

Distinct from the general "first bespoke internet-facing service" point already made in §5.9 — the specific things that need to be right, not just tested:

- **Webhook trust + idempotency**: the Stripe webhook (§5.9) is, from the outside, a "tell me this person paid" endpoint. Signature verification against Stripe's signing secret is the entire security boundary between "real payment" and "anyone who finds the URL" — this needs to be correct from the first deploy, not hardened later. **Idempotency is the other half:** Stripe retries webhooks on failure, so provisioning must be keyed on the Stripe event id — a duplicate `checkout.session.completed` event must not create a second account. Signature verification without dedup is still a free-account vector via retry.
- **Provisioning-session token lifecycle**: what happens if a user disconnects mid-flow, before or after paying but before completing signup? Resolved for MVP: **start over** — nothing is provisioned until payment is confirmed, so a lost mid-flow session costs the user a Stripe refund-or-retry, not a dangling account. A TTL on unclaimed paid sessions still applies; build resumption only if refunds become a real support queue.
- **Provisioning privilege**: the service that runs this flow ends up needing whatever privilege actually creates OS accounts and mailboxes — which makes it a meaningfully more sensitive process than anything else on the box apart from `sshd` itself. Resolved: **narrowly-scoped `sudo` rules** for exactly the provisioning commands it needs — more work to set up, but the whole trust boundary of the first bespoke service rests on it.

## 7. MVP v1 Scope

**In scope:**
- Single-server OpenBSD install (via the snapshot approach, §6.1), fully described in Terraform
- Mail: OpenSMTPD + Dovecot + Maildir + SPF/DKIM/DMARC + `spamd` + rspamd/sieve, dogfooded on Oliver's own address first
- Static hosting: `httpd` + `acme-client` + `ftpd` chroot upload
- Gemini hosting: `gmid`, reusing the FTP upload path
- Git/`pass` hosting: `git-shell`, SSH-key-only, per-user isolated repos
- Backups: nightly encrypted `rsync`, with a tested (not just assumed) restore procedure
- Automated abuse-detection scripts with an external notification channel (§5.8) — in scope alongside the mail stack, since thresholds are best tuned before real users generate false-positive noise
- External health monitoring (§6.11, layer 1) — cheap, and shortens detection time for any outage from day one
- Acceptable use policy and a minimal privacy policy sufficient to onboard real users under GDPR
- Account deletion: `scripts/del-user.sh` + the 40-day grace / admin-mediated deletion flow (§5.9) — GDPR erasure is not deferrable past onboarding real users
- Public `kyriakon-infra` repo with the structure in §5.6, published in parallel with the build rather than after
- Individual pricing tier only: flat £20/yr
- kyriakon-site content promotes Kleio as the recommended password manager for `pass` repos hosted here (§3) — a copy/content requirement, not a technical one

**In scope (resolved):**
- Onboarding & account portal — HTTP signup form + Stripe Checkout (§5.9.1). The platform's first bespoke, internet-facing, auth-handling service; §6.14 lists the trust-surface items that must be right before it ships. Manual/admin provisioning via `scripts/add-user.sh` remains the fallback if this slips, but it is not the plan.

**Fast-follow (after MVP, once mail/static/Gemini/git are stable and dogfooded):**
- SSH TUI signup/login (§5.9.2) and the embedded-in-Kleio experience (§3). Until then, Kleio connects to `git-shell` as any other SSH remote (§5.9).

**Deferred past MVP:**
- CGI support (`slowcgi`) — real feature demand should exist before taking on its resource-limiting and sandboxing burden
- Own-domain tier (~£50/yr, for parishes/businesses wanting `secretary@theirparish.org`) — same shared infrastructure, isolated mailboxes; straightforward to add once the individual tier is proven, not before
- Fully separate managed-VPS tier (~£150+/yr) — explicitly a later/premium option, not part of early build
- Any reconsideration of shell access (§6.7)
- Hidden-primary DNS (`nsd` + Hurricane Electric + Cloudflare CNAME/Partial setup, §5.7/§6.13) — Phase 4 hardening; Cloudflare-as-sole-provider is fine through MVP and beta
- Warm-standby VPS (§6.11, layer 4) — cost-gated; worth having before the platform becomes critical infrastructure for a parish (§6.6). Secondary MX (layer 3) is no longer deferred — it's in Phase 3, before the closed beta.
- Acting as DNS host for parish-owned domains (§6.13) — MVP own-domain tier requires only documented DNS-record instructions, not zone hosting

**Rationale**: this scope proves the two things that actually determine whether kyriakon.net is viable — that mail can be run reliably and trustworthily on a self-managed OpenBSD box, and that the no-shell service set (static/Gemini/git) is enough to be genuinely useful — without taking on CGI's sandboxing burden or multiple pricing tiers before there's a single tier's worth of real users to learn from.

## 8. Team & Working Model

This is currently a **solo build** — unlike Kleio, there's no stated engineering co-contributor for kyriakon.net. Marios and parish contacts are named as closed-beta *users* (§9, step 10), not as contributors to the infra repo itself. Marios does hold the second root credential for continuity (§6.6). If that changes, task allocation should follow the same skill/dependency-line approach used on Kleio (see Kleio's proposal §8) rather than an even split — but nothing in this proposal assumes it will.

Given the solo, infrastructure-heavy nature of the work, AI-agent assistance (via the same omp + skills workflow already running on Kleio) is likely to carry a larger share of routine work here than on Kleio — provisioning scripts, config file scaffolding, docs — precisely because there's no second pair of human eyes on this repo by default. §11 below covers how to adapt that workflow's guardrails accordingly.

## 9. Phased Plan

Numbering follows the handover doc's "Immediate Next Steps," grouped into phases; no time estimates are given since this is a part-time solo effort running alongside the job search and Kleio.

**Phase 1 — Foundations**
1. Install OpenBSD via the snapshot approach (§6.1) — snapshot-first, no autoinstall-response-file spike
2. Get OpenSMTPD + Dovecot + Maildir working for Oliver's own email — dogfooding surfaces deliverability and config problems before any other user is exposed to them
3. Draft the acceptable-use policy (shapes quota defaults §6.5 and abuse-response tooling §6.4)

**Phase 2 — Core services**
4. Static landing page on kyriakon.net via `httpd` + `acme-client`
5. Test `ftpd` chroot with a static-site upload as a non-root user
6. Set up `gmid`, serve a basic Gemini capsule
7. Configure `git-shell`, test a `pass` repo over SSH end-to-end

**Phase 3 — Governance & onboarding readiness**
8. Finalize the acceptable-use policy (drafted in Phase 1), minimal privacy policy, and pricing page
9. Stand up the secondary MX / backup mail relay (§6.11, layer 3) before the closed beta begins — including its PTR record and SPF inclusion, so relayed mail passes the primary's checks
10. Closed beta with 3–5 trusted users (Marios, parish contacts)

**Phase 4 — Public infra & hardening (parallel throughout, not a later phase)**
11. Begin the public Terraform/config repo (`kyriakon-infra`) in parallel with Phase 1–3 work, not after it — the "audit us" principle in §2 loses most of its value if the repo only appears once the platform is already live
12. Hidden-primary DNS migration (`nsd` + Hurricane Electric + Cloudflare CNAME setup, §5.7/§6.13) — free, pure config, lowers the public DNS answer surface; pulled ahead of CGI
13. CGI support (phase 2 feature, §7) — real sandboxing and resource-limiting work with no feature demand yet, so it lands last in Phase 4
14. Warm-standby VPS (§6.11, layer 4) — cost-gated; deferred until real user volume justifies roughly doubling the storage cost

Automated abuse monitoring and external health monitoring (§5.8, §6.11 layer 1) are MVP-scope (§7) and land in Phase 1 alongside the mail stack, not here — worth flagging since they're easy to mentally file under "hardening" along with the Phase 4 items above, but the rationale in §5.8 is specifically why they shouldn't wait.

## 10. Open Questions

1. ~~Whether the autoinstall-response-file path (§6.1, option 1) is worth a time-boxed spike...~~ — resolved: snapshot-first, no spike (§6.1).
2. ~~What mail warm-up plan (§6.2) is realistic...~~ — resolved: no formal warm-up campaign; organic dogfooding + beta volume is the warm-up, with a pre-first-send IP reputation check (§6.12).
3. ~~Who else, if anyone, should have emergency root access for continuity purposes (§6.6)...~~ — resolved: Marios (Kleio co-contributor), no one else for now (§6.6).
4. ~~Whether the acceptable-use policy should be drafted now (Phase 3) or earlier...~~ — resolved: draft it now (Phase 1), finalize wording in Phase 3 (§9).
5. ~~Whether ticket/epic breakdown should happen now via `/to-tickets`...~~ — settled: Oliver runs `/to-tickets` locally himself.
6. ~~Whether `git-shell` alone is sufficient...~~ — resolved, see §6.8.
7. ~~Which static/Gemini directory layout (§6.9) to standardize on...~~ — resolved: sibling `www/` + `gemini/` per user (§6.9).
8. ~~Whether §6.11's single-point-of-failure mail-loss risk is an accepted MVP-scale trade-off...~~ — resolved: pull the secondary MX (layer 3) into Phase 3, before the closed beta (§6.11).
9. ~~Which external notification channel (§5.8) to use...~~ — resolved: Healthchecks.io, doubling as the external health-monitoring ping target (§5.8, §6.11 layer 1).
10. ~~Whether the Phase 4 DNS migration (§5.7/§6.13) is worth doing on its own schedule ahead of other Phase 4 items...~~ — resolved: pull it ahead of CGI; DNS first, CGI last (§6.13, §9).
11. ~~Whether a secondary MX (§6.11, layer 3) should move earlier than Phase 4...~~ — resolved: yes, into Phase 3, before the closed beta (§6.11, §9).
12. ~~Whether the onboarding service (§5.9) ships as part of MVP v1 or as a fast-follow...~~ — resolved: HTTP signup + Stripe in MVP, SSH TUI fast-follow (§5.9, §7).
13. ~~Payment processor: Stripe is the natural default...~~ — resolved: **Stripe Checkout** (§5.9.1). Alternatives were checked and rejected: BTCPay Server (open-source, self-hosted, but Bitcoin/Lightning-only — no cards, which excludes the target audience) and GNU Taler (open-source but no card checkout and effectively no ecosystem). No open-source card processor exists that removes PCI scope the way Stripe's hosted Checkout does. GoCardless Direct Debit is cheaper for UK recurring but UK-bank-only and no cards. Stripe's fee difference on a £20/yr subscription is pennies; the philosophy concern is real but there is no card-capable open-source answer, and hosted Checkout's PCI-scope removal is itself a security win.
14. ~~Provisioning-session resumption (§6.14)...~~ — resolved: start over for MVP, with a TTL on unclaimed paid sessions; resumption only if refunds become a real queue (§6.14).
15. ~~Provisioning privilege model for the onboarding service (§6.14)...~~ — resolved: narrowly-scoped `sudo` rules for exactly the provisioning commands (§6.14).
16. ~~Whether `acme-client` supports DNS-01...~~ — **resolved, confirmed against the current OpenBSD man page**: `acme-client` implements HTTP-01 only, no DNS-01 support at all (this has been true since it first shipped in OpenBSD 6.1, not a version gap). The wildcard `*.kyriakon.net` cert (§5.2) is routed through Cloudflare's proxy/Universal SSL, as already flagged as the fallback.
17. ~~Mail-at-rest encryption mechanism (§2, §5.1)...~~ — resolved: **both** — full-disk `softraid` crypto *and* per-message Dovecot `mail_crypt` (§2, §5.1).
18. Optional `pass` key backup in the SSH TUI, unlocked by a recovery code (§5.9.2) — an idea, not a decision; `pass` data-loss/recovery is Kleio's subject. Carried as a note to Kleio's scoping.
19. ~~Account deletion / GDPR erasure path...~~ — resolved: admin-mediated, 40-day grace with user notification, `scripts/del-user.sh` (§5.9).
20. ~~Default storage quota (§6.5)...~~ — resolved: **5 GB** per account across mail + web + git (§6.5); upgradeable later, not reducible without migration.

---

## 11. Working with AI agents on this repo (omp setup)

Kleio's `docs/planning/agent-setup.md` and `AGENTS.md` are a solid direct template for `kyriakon-infra` — same omp + DeepSeek v4 Pro/Flash + `mattpocock/skills` + personal-extensions stack — but three things about this repo change how it should be configured, not just copied:

- **The blast radius of a bad agent action is different.** A bad Rust PR on Kleio fails CI. A bad `pf.conf` or `sshd_config` change on kyriakon-infra, applied without review, can lock out real users' mail or open the box up. The workflow should lean harder on "propose, don't apply" for anything touching `pf.conf`, `sshd_config`, `smtpd.conf`, or the Terraform that provisions the live box.
- **Secrets discipline matters more here than on Kleio.** Kleio's `betterleaks`-style pre-commit hook is necessary but not sufficient when the repo will, over time, accumulate people who've seen its structure — DKIM keys, TLS keys, and any admin tooling need to never even be *proposed* by an agent, not just caught by a hook after the fact.
- **There's no second human reviewer by default** (§8), so the guardrails that on Kleio exist partly for Marios's benefit need to fully carry the weight here.

### 11.1 Reuse as-is from Kleio

- omp itself, `~/.omp/agent/models.yml` and `config.yml` (DeepSeek v4 Flash as `default`/`smol`/`commit`, DeepSeek v4 Pro as `slow`/`plan`) — no changes needed, this repo doesn't demand a different model tier.
- `mattpocock/skills` marketplace install — `wayfinder`, `grilling`, `domain-modeling`, `code-review`, `to-spec`, `to-tickets`, `implement` all map cleanly onto infra work; nothing here is Rust/TS-specific.
- The `docs/planning/` workflow (`research/` → `specs/` → GitHub Issues via `/to-tickets`, with a `Spec:` line back to the spec file) and the `docs/decisions/` ADR pattern this proposal itself follows.
- **`guardrails`** — keep it, and if anything, extend it: on Kleio it blocks edits to `AGENTS.md`; here, add `openbsd/etc/pf.conf`, `openbsd/etc/ssh/sshd_config`, and anything under a `secrets/`-style path (even if that path is meant to stay empty/gitignored) to the same block-list, so an agent can *propose* a diff but never *write* to the security-critical files directly.

### 11.2 Adapt for kyriakon-infra

- **`AGENTS.md`** — same shape as Kleio's (orientation → shared vocabulary pointer → build/test → code style → tickets → git rules → "Never" list → skills), but the "Never" section should be infra-specific and explicit, e.g.: never run `terraform apply` or `terraform destroy` against the live workspace without it being the human-approved step of an already-reviewed PR; never write real secrets, key material, or user data into any tracked file, including as "example" values in docs; never modify `pf.conf`/`sshd_config` outside a PR the human has explicitly asked for and will review line-by-line.
- **`CONTEXT.md`** — a shorter glossary than Kleio's, but still worth having so agent output uses consistent terms: e.g. distinguishing "shell-less user" (the standard tier, §2) from any future shelled tier; "own-domain tier" vs "individual tier" (§7); "dogfooding" as the specific Phase 1 step (§9.3) rather than a generic synonym for testing.
- **`docs/decisions/`** — this proposal is ADR #0, following Kleio's pattern of keeping the founding proposal itself in `docs/decisions/`. The concrete near-term ADRs to expect out of §10's open questions: the OpenBSD-install-automation decision once §6.1's spike concludes, and the acceptable-use-policy scope once §10.5 is resolved — both are exactly the kind of "why does the config look like this" decision `docs/agents/domain.md`'s pattern is meant to catch.
- **`docs/agents/issue-tracker.md`** — reusable close to verbatim; the `gh`-based conventions don't change. The triage-label mapping table (`docs/agents/triage-labels.md`) also carries over unchanged.

### 11.3 One new consideration Kleio doesn't have: a "propose-only" tier for host changes

Kleio's `guardrails` extension is binary — an agent either can or can't touch a file. For kyriakon-infra, it's worth asking `mattpocock/skills`' `/implement` skill (or a small custom extension alongside `ponytail`/`caveman`/`guardrails` in the `omp-extensions` marketplace repo) to draft any `pf.conf`/`sshd_config`/Terraform-apply-adjacent change as a PR diff *only*, with the live-box-affecting command (the actual `terraform apply`, the actual `scp` of a new `sshd_config` to the box) left as a manual step documented in the PR body rather than something the agent runs itself — even inside an otherwise-trusted session. This is a small addition to what's already built for Kleio, not a different toolchain.

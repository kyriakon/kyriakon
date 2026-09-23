# 0001 Domain registrar and delegation

Status: accepted
Date: 2026-09-11
Amended: 2026-09-22, extended to cover `kyriakon.com`, with the consolidation marked provisional
pending `kyriakon-infra` issue #70
Amended: 2026-09-22, registrar re-assessment after #70 concluded: registry lock state, the non-EU
registrar constraint, and registrar concentration

## Context

The proposal's section 5.7 and section 6.13 require `nsd` on the box as a hidden primary, with
Hurricane Electric's free secondary answering public queries, so the registry delegation for
`kyriakon.net` must be `ns1.he.net` through `ns5.he.net`. Section 6.13 already rules out Cloudflare
in the DNS path: its secondary DNS and partial-setup products are priced for enterprises, and the
zone should be git-auditable in `kyriakon-infra` rather than living in a third-party dashboard.

The domain is registered with Cloudflare Registrar. At the registry, before the transfer:

```
registrar:    Cloudflare, Inc.
nameservers:  KAYLEIGH.NS.CLOUDFLARE.COM, LEONIDAS.NS.CLOUDFLARE.COM
DS records:   none
expires:      2028-11-17
```

Cloudflare's documentation states that a domain acquired from Cloudflare Registrar uses Cloudflare
nameservers, and that using a different DNS provider requires transferring the domain away from
Cloudflare. The registrar cannot express the delegation the architecture requires, which makes this
a hard blocker rather than a configuration problem.

Two attempts are worth recording so they are not repeated. Adding an `NS` record for the apex
pointing at `ns1.he.net` inside a zone is a subdomain delegation within that zone, not a change of
registry delegation, so it cannot move authority. Terraform adds nothing either: no first-party
Porkbun provider exists, and the community providers carry a `destroy` that deletes a registration,
in exchange for managing a value that changes once.

A second domain is in scope for the same reason. `kyriakon.com` is a defensive registration: it
exists so that nobody else can use the name, not to serve anything. It is delegated to Hurricane
Electric like the primary, it answers a blanket denial record set, and the box redirects it to
`kyriakon.net`. That infrastructure is already live, which is why the registrar question applies to
it now rather than being a future concern.

## Decision

Transfer `kyriakon.net` out of Cloudflare Registrar to Porkbun, and set the registry nameservers to
`ns1.he.net` through `ns5.he.net` there. Porkbun holds the delegation and nothing else, while the
zone stays in `openbsd/etc/nsd/` in `kyriakon-infra`, served by the box and by HE.

Porkbun permits arbitrary external nameservers, supports WebAuthn and U2F hardware keys as well as
passkeys for account login, prices transfers and renewals flat with free WHOIS privacy, and
publishes an API if the delegation ever needs scripting.

Both domains consolidate at Porkbun. `kyriakon.com` was registered at Dynadot on 2026-09-17, so it
sits inside ICANN's 60-day post-registration transfer prohibition, which lifts around 2026-11-16. It
moves after that.

The transfer is not automatic and needs three things done deliberately: the `client transfer
prohibited` status removed, an auth code obtained from Dynadot, and the authorisation email actioned
by the registrant contact. The last is the quiet failure mode of a planned transfer, and it cannot be
checked from outside the account: registry RDAP redacts registrant contacts and exposes only the
registrar handle.

That consolidation is provisional rather than settled. `kyriakon-infra` issue #70 is scoping
resistance to a state-level adversary as a product property, and one of its open questions is
whether jurisdiction diversity across registrars and registries is warranted. If it concludes that
it is, that conclusion supersedes this paragraph, and INWX remains the EU-jurisdiction alternative
named above.

Alternatives and why they lost. INWX is equally capable and remains the alternative if EU
jurisdiction is wanted later. Hetzner's documented DNS configurations all include Hetzner
nameservers, and it would put the domain, the VPS and the backup storage behind one account.
Njalla holds domains in its own name, which defeats auditability and continuity. Gandi has drifted
on pricing and policy since its acquisition. GoDaddy offers SMS as its only second factor.

## Consequences

The registrar is no longer the DNS provider. The Cloudflare DNS zone remains as a rollback path,
because reverting the nameservers at Porkbun to Cloudflare's pair restores the previous state, and
it is deleted once HE has served the zone without incident.

Porkbun preserves the nameservers already assigned to a domain during a transfer, so Cloudflare
keeps answering until the delegation is deliberately changed, and the mail records become public at
the moment HE is authoritative.

Account security requirements: a hardware key or passkey as the second factor on the registrar
account, and registrant contact details edited only deliberately, because ICANN prohibits a transfer
for 60 days after a registrant change.

DNSSEC is off today. If HE ever signs the zone, the DS records have to be published at Porkbun.

The transfer completed on 2026-09-11. The registry now shows registrar Porkbun LLC, nameservers
`NS1.HE.NET` through `NS5.HE.NET`, status `client delete prohibited` and `client transfer
prohibited`, and expiry 2029-11-17 after the year a transfer adds. Public resolvers return HE's five
nameservers, and the zone answers through them: SOA serial `2026082701`, `10 mail.kyriakon.net.`,
the box's IPv4 and IPv6 for the apex, `mail` and the wildcard, SPF, the DKIM placeholder, and DMARC.

`kyriakon.com` is delegated to HE on the same five nameservers and is answering publicly: a null MX
at both the apex and the wildcard per RFC 7505, `v=spf1 -all` at both, empty-key DKIM denial for any
selector, and the apex pointing at the box so the redirect and its certificate work. The wildcards
are the point. A forger can choose any name under a domain, not just the apex, so a per-name denial
would leave the rest of the namespace open.

Its registry state differs in one respect worth closing. It is at Dynadot3804 LLC, expires
2027-09-17, and carries `client transfer prohibited` but not `client delete prohibited`, where
`kyriakon.net` carries both. For the one domain whose entire purpose is that nobody else can use the
name, an accidental deletion is the failure that defeats the purpose. Either Dynadot adds the status
or the transfer does, and that should be confirmed rather than assumed.

Neither lock is what actually keeps a defensive domain. Auto-renew against a reachable registrant
contact is, and `kyriakon.com` expires two years before `kyriakon.net` does.

## Amendment, registrar re-assessment (2026-09-22)

`kyriakon-infra` issue #70 has concluded, and it settles the provisional paragraph above in favour of staying: both gTLDs remain at Porkbun, and jurisdiction diversity is bought with a separate ccTLD rather than by moving either of them (ADR 0006). Three things the original record did not cover were assessed.

### The registry lock is in place on neither domain

Neither domain is registry-locked. Verisign's criterion is that all three of Update, Delete and Transfer carry a `server...Prohibited` status, and `kyriakon.net` carries `client delete prohibited` and `client transfer prohibited`, which are registrar-level and therefore a different guarantee. `kyriakon.com` carries `client transfer prohibited` alone, so it is weaker on the status that matters most for a defensive registration, which is accidental deletion.

Applying the lock is a registry action mediated by the registrar, and lifting it runs the same way in reverse: out-of-band verification with the registrant, with Dynadot quoting three to five business days in each direction. That asymmetry decides the order. Lock `kyriakon.net`, which is settled and is not moving. Do not lock `kyriakon.com` before its transfer, because a lock that must be lifted inside the transfer window turns a routine move into a support round trip against a deadline.

### A second registrar is required, and it has to be Swiss

Issue #70 requires the ccTLD to sit outside the European Union, and `.ch` is the chosen name. SWITCH accredits its registrars, and every non-EU registrar on that list is Swiss, so the second registrar will be a Swiss one. Porkbun cannot register `.ch` at all, which was checked against its own public pricing endpoint: 909 TLDs, `.ch` not among them. The candidates are INWX, if the contracting entity is the Zürich one rather than the `inwx.de` presence or the "INWX Inc." the site also names, and Infomaniak in Geneva. Both need confirming as permitting arbitrary external nameservers, because the delegation has to reach HE.

That supersedes the earlier framing of INWX as the EU-jurisdiction alternative. For this decision the property that matters is its Swiss presence, and which entity contracts is the thing to confirm rather than assume, since a United States corporation would defeat the purpose entirely.

### Concentration

Two names at one American registrar concentrate both on the same legal process, and that concentration is jurisdictional rather than corporate. Moving a gTLD to a second American registrar would spread the account risk and leave the jurisdictional exposure exactly as it was, so the answer is the ccTLD at a Swiss registrar rather than a shuffle among gTLDs.

### Still open

Whether Dynadot adds `client delete prohibited` to `kyriakon.com`, or the transfer to Porkbun does. It is the status whose absence matters most for the one domain whose only purpose is that nobody else can use the name, as the Consequences above already note.

## References

- Proposal sections 5.7 and 6.13, in `docs/decisions/kyriakon-net-project-proposal.md`
- [Cloudflare Registrar: transfer out](https://developers.cloudflare.com/registrar/account-options/transfer-out-from-cloudflare/)
- [Cloudflare DNS: nameservers](https://developers.cloudflare.com/dns/nameservers/)
- [Porkbun: transfer a domain in](https://kb.porkbun.com/article/56-how-to-transfer-a-domain-to-porkbun) and [nameserver import behaviour](https://kb.porkbun.com/article/117-will-my-nameservers-be-imported-during-a-transfer)
- `kyriakon-infra` issue #24 and pull request #52
- `kyriakon-infra` issue #70, scoping resistance to a state-level adversary, which governs whether
  registrar diversity supersedes the consolidation above
- `kyriakon-infra` `openbsd/etc/nsd/kyriakon.com.zone`, `nsd.conf`, `acme-client.conf`,
  `httpd.conf` and `scripts/deploy-nsd.sh`, where the defensive registration is implemented

Registry and delegation state for both domains was regenerated with:

```sh
for d in kyriakon.net kyriakon.com; do
  curl -fsS -H 'Accept: application/rdap+json' \
    "https://rdap.verisign.com/${d##*.}/v1/domain/$d" | jq -rc '.status'
done
dig +short NS kyriakon.com @8.8.8.8
```

Registry state in this record was regenerated with:

```sh
curl -fsS -H 'Accept: application/rdap+json' https://rdap.verisign.com/net/v1/domain/kyriakon.net
dig +short DS kyriakon.net @8.8.8.8
```

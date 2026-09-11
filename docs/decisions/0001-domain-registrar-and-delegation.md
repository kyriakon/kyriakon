# 0001 Domain registrar and delegation for kyriakon.net

Status: accepted
Date: 2026-09-11

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

## Decision

Transfer `kyriakon.net` out of Cloudflare Registrar to Porkbun, and set the registry nameservers to
`ns1.he.net` through `ns5.he.net` there. Porkbun holds the delegation and nothing else, while the
zone stays in `openbsd/etc/nsd/` in `kyriakon-infra`, served by the box and by HE.

Porkbun permits arbitrary external nameservers, supports WebAuthn and U2F hardware keys as well as
passkeys for account login, prices transfers and renewals flat with free WHOIS privacy, and
publishes an API if the delegation ever needs scripting.

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

## References

- Proposal sections 5.7 and 6.13, in `docs/decisions/kyriakon-net-project-proposal.md`
- [Cloudflare Registrar: transfer out](https://developers.cloudflare.com/registrar/account-options/transfer-out-from-cloudflare/)
- [Cloudflare DNS: nameservers](https://developers.cloudflare.com/dns/nameservers/)
- [Porkbun: transfer a domain in](https://kb.porkbun.com/article/56-how-to-transfer-a-domain-to-porkbun) and [nameserver import behaviour](https://kb.porkbun.com/article/117-will-my-nameservers-be-imported-during-a-transfer)
- `kyriakon-infra` issue #24 and pull request #52

Registry state in this record was regenerated with:

```sh
curl -fsS -H 'Accept: application/rdap+json' https://rdap.verisign.com/net/v1/domain/kyriakon.net
dig +short DS kyriakon.net @8.8.8.8
```

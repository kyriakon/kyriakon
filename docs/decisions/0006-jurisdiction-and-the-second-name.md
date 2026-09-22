# 0006 Jurisdiction and the second name

Status: accepted
Date: 2026-09-22

## Context

The platform holds `kyriakon.net` and `kyriakon.com`, both at Porkbun, and both American in substance: `.net` and `.com` are Verisign registries, and Porkbun is a United States registrar. The server is a Hetzner virtual machine in Germany and the backup repository sits on a Hetzner storage box in Germany as well. One jurisdiction therefore reaches the names and the machine, which is the shape the suppression adversary attacks.

A second name outside the European Union was assessed as a way to break that. Three candidates were considered.

Iceland, `.is`, was rejected on cost and on three registry constraints that work against the reason for holding a name in reserve. Every nameserver must be registered with ISNIC before a delegation resolves, which HE's shared anycast nameservers are not. NS record TTL must be at least 24 hours, enforced on exactly the records a hurried re-home would want short. A monthly automated compliance test can put the domain on hold with its DNS removed after eight weeks of failure. At roughly £50 to £60 a year against about £12, that is a poor trade.

Switzerland, `.ch`, costs about £12 a year, and SWITCH is the registry, so the name sits outside the European Union. SWITCH publishes its accredited registrar list, and every non-EU registrar on that list is Swiss. Porkbun cannot register `.ch` at all, verified against its own public pricing endpoint: 909 TLDs, `.ch` not among them.

## Decision

Hold `kyriakon.net` and `kyriakon.com` where they are, and move neither.

Acquire `.ch` defensively, as a second name, when paying members reach eight. That is a gate rather than a date, and it exists because the resolved cost model in the proposal's section 2, roughly £113 a year fixed across one VPS, one storage box and one domain, no longer describes the deployment: there are two boxes, and there will be two domains, with the restore machine absent from that figure entirely.

The registrar must be outside the European Union, and because SWITCH is the only route to `.ch` that means Swiss. The candidates are INWX, if the contracting entity is the Zürich one rather than the `inwx.de` presence or the "INWX Inc." the site also names, and Infomaniak in Geneva. Both need confirming as permitting arbitrary external nameservers, because the delegation has to reach HE.

A re-home, if one is ever attempted, is registry, registrar and hosting jurisdiction together or not at all. Any single part leaves the other two as the choke point, which is what makes a second name worth less than it first appears.

## Consequences

The second name does not defend the service. The server and the storage box are both in Germany, so an EU request takes the mail whatever the domain resolves through. What the acquisition buys is a delegation that can be re-pointed without the current host's cooperation, and the removal of the name from a European legal process. The lever for the platform itself is hosting jurisdiction, which ADR 0007 defers on cost.

The gate is counted in paying members rather than revenue because the fixed costs are paid from members, and the count is checkable against the payment state the portal holds on the box rather than against a payment processor.

The Iceland assessment is recorded here so that it is not run again. The monthly compliance test was the deciding constraint rather than the price, since a registry that can remove DNS on a failed test of its own design is a registry with a second suppression path.

Two names at one American registrar is a concentration, and the concentration is jurisdictional rather than corporate, so the answer to it is the ccTLD rather than a second American registrar.

## References

- ADR 0001, domain registrar and delegation, including the registry lock assessment
- ADR 0007, provider posture and key custody
- Glossary: `docs/CONTEXT.md`, reading adversary and suppression adversary
- `kyriakon-infra` issue #70 and its triage comment
- Proposal section 2, the fixed cost model

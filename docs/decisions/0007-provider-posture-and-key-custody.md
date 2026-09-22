# 0007 Provider posture and key custody

Status: accepted
Date: 2026-09-22

## Context

The platform runs on one Hetzner virtual machine in Germany, and its nightly encrypted restic repository sits on a Hetzner storage box, also in Germany. The firmware, the hypervisor and the management engine are not the platform's to inspect or control, and a memory snapshot taken from outside the guest sees what the guest holds at that moment.

Message bodies are PGP ciphertext and the platform holds no key to them, which ADR 0002 and ADR 0003 make structural by keeping classification off the mail box and refusing third-party inference. The machine's own secrets are a different matter, since the queue key, the DKIM signing key and the restic password all rest on it.

Boot-derived custody was considered for the restic password, holding it only in memory and deriving it from a passphrase entered at boot. It was rejected for two reasons. `rc.d` has no terminal, the same constraint that already put the mail queue key at `/etc/mail/queue.key` rather than behind `getpass(3)`, so there is nothing to type a boot passphrase into. Restic cannot write an encrypted repository without the key that decrypts it, so a machine that backs itself up holds the key to its own archive by construction, and a snapshot of a running VM captures RAM, which leaves memory-only custody answering the powered-off disk image and nothing beyond it.

## Decision

Stay on Hetzner, and state the hypervisor ceiling as a named limit in the threat model rather than leave it implied.

Minimise what sits there. Text is not retained beyond the 90-day window (ADR 0005) and content classification stays off the mail box (ADR 0002), so the machine is worth less as a target than a mail server normally is.

The restic password stays on the machine. The mitigation is a separate offline copy whose key the machine never holds, refreshed quarterly and timed with the quarterly rehearsal so the rehearsal runs against the freshest copy.

The offline copy uses its own repository and passphrase, on the external SSD, with the passphrase stored on the same disk as the copy. That drive already holds raw `~/.ssh`, `~/.gnupg`, `~/.password-store` and `~/.wallets`, and the chezmoi age key is already written down on paper as the recovery path for that drive, so one more passphrase alongside them adds nothing to an exposure that exists. The change that would matter is encrypting the volume, which belongs to the dotfiles repository rather than here.

The regime for the copy is quarterly against the nightly cadence of the online repository, so an offline restore loses at most a quarter and the online restore at most a day.

Owned hardware stays last, and only on an observed provider action against the account: a suspension, a disclosure demand, or a demand to migrate. A second host in a second jurisdiction is the item that would answer the provider class, and it is deferred on cost rather than dismissed.

## Consequences

The platform's availability rests on one provider in one jurisdiction. This ADR is where that is admitted, and the threat model states it as a limit rather than leaving it as a gap.

The offline copy is the only copy that survives a compromise of the running machine, since every other copy sits behind the password that machine holds.

Recovery loss is bounded by the nightly cadence for the online repository and by a quarter for anything only the offline copy holds. Both are stated ceilings rather than accidents.

Sharing a disk between the passphrase and the copy is a deliberate choice with a known ceiling: whoever holds the drive holds everything on it. Volume encryption is the upgrade path, and it is tracked in the dotfiles repository where the drive is configured.

## References

- ADR 0002, content classification stays off the mail box
- ADR 0005, text is not retained beyond the 90-day application purge
- ADR 0006, jurisdiction and the second name
- `kyriakon-infra/docs/threat-model.md`
- `kyriakon-infra` issue #70 and its triage comment
- Dotfiles repository: `backup.sh`, the SSD copy, and the paper copy of the age key

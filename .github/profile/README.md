<div align="center">

# ☧ Kyriakon

**Secure email, static web hosting, and `pass` repositories.**
*Principled hosting, run by Orthodox Christians.*

</div>

---

Kyriakon.net is a low-extraction, no-shell hosting platform for the Orthodox Christian
community - mail, static/Gemini sites, and `pass`-compatible git repos - running on
OpenBSD. Every non-secret piece of infrastructure config is published here on GitHub.

> **Audit us.** Secure email, server configuration you can audit yourself, and Orthodox
> admins you can actually meet.

---

## Projects

| | |
|:---|:---|
| **[kyriakon-infra](https://github.com/kyriakon/kyriakon-infra)** | Terraform + OpenBSD configuration for the platform itself - the implementation half |
| **[kyriakon-onboard](https://github.com/kyriakon/kyriakon-onboard)** | SSH TUI signup/login and Stripe-webhook receiver - the first bespoke, internet-facing service |
| **[kyriakon-site](https://github.com/kyriakon/kyriakon-site)** | The static kyriakon.net marketing/landing site |
| **[kyriakon](https://github.com/kyriakon/kyriakon)** | Cross-cutting decisions and shared vocabulary - the source of truth |
| **[Kleio](https://github.com/kyriakon/kleio)** | The pass-compatible password manager we recommend for communal security |

## Principles

- **Audit us, don't trust us.** All non-secret configuration is public, from day one.
- **No shell access.** Every service maps to one OpenBSD base-system daemon in a
  restricted mode; standard accounts are shell-less.
- **Zero-access mail.** The platform holds only public keys and encrypts on ingress -
  a disclosure order yields ciphertext, not mail.
- **Accessible, not extractive.** Priced at real break-even plus a small buffer.
- **Built for longevity.** Open-source stewardship over growth.

---

<div align="center">

**Kyriakon.net** - [kyriakon.net](https://kyriakon.net) - [hello@kyriakon.net](mailto:hello@kyriakon.net)

</div>

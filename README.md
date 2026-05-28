# omegaminds-mta-sts

This repository hosts the **MTA-STS policy file** for the `omegaminds.com` mail
domain. The policy is published to the Internet via GitHub Pages at:

> **`https://mta-sts.omegaminds.com/.well-known/mta-sts.txt`**

It exists as a piece of Omega Minds, LLC's CMMC compliance program — specifically,
the external-mail boundary hardening for the Google Workspace tenant that handles
Federal Contract Information (FCI). See [CMMC / compliance context](#cmmc--compliance-context)
below.

---

## What MTA-STS is

**MTA-STS (Mail Transfer Agent Strict Transport Security)** is an Internet
standard defined in [RFC 8461](https://datatracker.ietf.org/doc/html/rfc8461). It
lets a domain publish a machine-readable policy that tells *other* mail servers
two things:

1. Inbound mail to this domain MUST be delivered over TLS.
2. The receiving mail servers MUST present a publicly-trusted certificate whose
   name matches one of an enumerated set of expected MX hostnames.

A sending mail server (one delivering mail TO `omegaminds.com`) discovers the
policy in two steps:

1. It looks up the DNS TXT record at `_mta-sts.omegaminds.com`. If present, the
   record's `id=` value tells the sender that a policy exists and identifies the
   policy version.
2. It fetches the policy file over HTTPS from
   `https://mta-sts.<domain>/.well-known/mta-sts.txt` — i.e., the URL above. The
   HTTPS connection itself must validate against a publicly-trusted CA; there is
   no "ignore certificate errors" fallback in the protocol.

Together these prevent a network attacker from downgrading inbound mail delivery
to cleartext (a STARTTLS strip attack) or from redirecting it to an
attacker-controlled mail server with a self-signed certificate.

## Why this repository is public

MTA-STS policy files are **intentionally world-readable** by protocol design
(RFC 8461 §3.2). They MUST be reachable by any mail server on the Internet
without authentication, and they intentionally contain no secrets — only the
domain's expected MX hostnames and a TLS-enforcement preference. There is no
security value in restricting access, and a private repository could not serve
the file at all. The repo's public visibility is a deliberate,
protocol-mandated choice, not an oversight.

## How this repository serves the file

- **Content** lives in [`/.well-known/mta-sts.txt`](.well-known/mta-sts.txt) on
  the `main` branch.
- **GitHub Pages** publishes the `main` branch root as a static site.
- The [`CNAME`](CNAME) file tells GitHub Pages to serve the site under the
  custom domain `mta-sts.omegaminds.com`.
- DNS at the registrar (GoDaddy) points `mta-sts.omegaminds.com` (CNAME) to
  `ericpsu.github.io`.
- GitHub Pages automatically issues and renews a Let's Encrypt TLS certificate
  for the custom domain. There is no manual certificate maintenance.

## Current policy

```
version: STSv1
mode: testing
mx: aspmx.l.google.com
mx: *.aspmx.l.google.com
max_age: 86400
```

Field-by-field:

- **`version: STSv1`** — the only currently-defined MTA-STS version.
- **`mode: testing`** — senders should evaluate the policy and *report* failures
  via the companion TLS-RPT DNS record, but should NOT refuse delivery if the
  policy is violated. This is the first stage of an MTA-STS rollout: it allows
  Omega Minds to observe how real-world senders interact with the policy
  without risking inbound mail loss from a misconfiguration.
- **`mx: aspmx.l.google.com`** and **`mx: *.aspmx.l.google.com`** — Google
  Workspace's MX hostnames. The wildcard `*.aspmx.l.google.com` covers the four
  backup MX targets (`alt1` through `alt4`).
- **`max_age: 86400`** — senders cache the policy for one day. RFC 8461 §5
  recommends `max_age >= 86400` in production; the testing-mode rollout uses
  this minimum so that policy changes propagate quickly during the observation
  period.

A future migration to **`mode: enforce`** is planned once testing-mode evidence
shows no benign senders are failing the policy. That promotion will be tracked
through Omega Minds' change-control process and will raise `max_age`.

## Updating the policy

1. Edit [`.well-known/mta-sts.txt`](.well-known/mta-sts.txt) and commit to
   `main`. GitHub Pages will redeploy automatically within a few minutes.
2. **Bump the `id=` value** in the `_mta-sts.omegaminds.com` DNS TXT record at
   the registrar. The `id` is an opaque string; convention is a `YYYYMMDDNN`
   datestamp (for example, `2026052701`). Senders cache the policy keyed by
   `id`; without an `id` change, senders may continue using the old cached
   version until `max_age` elapses.
3. Record the change in the Omega Minds compliance repository
   (`EricPSU/CMMC`, private) under `evidence/changes/`, citing this repository
   and the GitHub commit SHA so the change is auditable end-to-end.

## Companion DNS records

These records live at the registrar on `omegaminds.com`. They are not stored in
this repository, but they are part of the same control implementation and
listed here for operator reference.

| Record | Type | Value | Purpose |
|---|---|---|---|
| `mta-sts.omegaminds.com` | CNAME | `ericpsu.github.io` | Points the policy hostname at GitHub Pages |
| `_mta-sts.omegaminds.com` | TXT | `v=STSv1; id=<datestamp>` | Signals to senders that an MTA-STS policy exists; `id` identifies the policy version |
| `_smtp._tls.omegaminds.com` | TXT | `v=TLSRPTv1; rua=mailto:dmarc-reports@omegaminds.com` | TLS Reporting ([RFC 8460](https://datatracker.ietf.org/doc/html/rfc8460)); collects TLS-failure reports from senders |

## CMMC / compliance context

Omega Minds, LLC is implementing
[CMMC (Cybersecurity Maturity Model Certification)](https://dodcio.defense.gov/CMMC/)
**Level 1** for Federal Contract Information (FCI), with a forward-looking
**Level 2-capable Google Workspace enclave** for future Controlled Unclassified
Information (CUI) work. This MTA-STS policy is part of the Workspace tenant's
external-mail boundary hardening, addressed in the System Security Plan under:

- **Primary:** **SC.L2-3.13.1** — *Monitor, control, and protect organizational
  communications (i.e., information transmitted or received by organizational
  systems) at the external boundaries and key internal boundaries of
  organizational systems.* (NIST SP 800-171 Rev 2 §3.13.1)
- **Secondary (load-bearing at L2):** **SC.L2-3.13.8** — *Implement
  cryptographic mechanisms to prevent unauthorized disclosure of CUI during
  transmission unless otherwise protected by alternative physical safeguards.*

The authoritative System Security Plan, evidence captures, and change records
are maintained in a separate private repository (`EricPSU/CMMC`). This public
repository contains only the policy file required by RFC 8461, the supporting
custom-domain configuration, and this README.

## Operational notes

- **Certificate renewal:** Fully automatic via GitHub Pages. No human action
  required and no expiration to track.
- **Repository deletion or rename will break inbound mail security** for
  `omegaminds.com` senders that have cached the policy, until DNS is repointed.
  Do not rename or delete this repository without coordinating a DNS update at
  the registrar.
- **Repo access:** Read access is intentionally public; write access is
  restricted to Omega Minds administrators.
- **Verification:** to confirm the policy is serving correctly, the URL above
  should return the five lines under [Current policy](#current-policy) over a
  valid TLS connection. Any browser error about an untrusted certificate
  indicates a misconfiguration that will cause sending mail servers to ignore
  the policy.

## License

This repository contains a configuration artifact specific to Omega Minds, LLC.
It is not reusable software. No license is granted; reuse is neither expected
nor encouraged.

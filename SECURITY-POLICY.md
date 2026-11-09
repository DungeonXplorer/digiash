# DigiAsh Information Security Policy

| | |
|---|---|
| Version | 1.0 |
| Last reviewed | October 2026 |
| Owner | DungeonXplorer (maintainer) |
| Next review | October 2027, or sooner after a significant change |

> **Certification status.** This policy is *inspired by* the structure and control themes of ISO/IEC 27001:2022. DigiAsh is **not** certified against ISO/IEC 27001 or any other standard, and has not been independently audited. References to the standard describe the approach taken, not a claim of conformity.

## 1. Purpose

This policy sets out how DigiAsh protects the information it handles: user accounts, the API keys users store, the searches they run, and the service itself. It records what is in place today and what is planned, so that users and reviewers can see how security decisions are made.

## 2. Scope

The policy covers:

- the DigiAsh web application (the website, its API and the dashboard);
- the infrastructure it runs on: Cloudflare Workers, Cloudflare D1 (database), Cloudflare KV (key storage) and Cloudflare Turnstile;
- the source code and its repository;
- the third-party services DigiAsh connects to on a user's behalf;
- the maintainer's accounts and devices used to build and operate the service.

## 3. Principles

1. **Security by design.** Security requirements are considered when a feature is designed, not added afterwards.
2. **Least privilege and data minimisation.** DigiAsh collects only what the service needs and gives each component only the access it needs.
3. **Defence in depth.** No single control is relied on alone; for example, rate limits, lockouts and bot checks all protect sign-in.
4. **Untrusted input.** Everything received from users and from third-party providers is treated as untrusted and validated or escaped before use.
5. **Transparency.** Controls that are planned but not yet in place are labelled as such in this document.

## 4. Roles and responsibilities

| Role | Responsibilities |
|---|---|
| Maintainer (DungeonXplorer) | Owns this policy; designs, builds and operates the service; manages secrets and infrastructure access; handles security reports and incidents; reviews this policy. |
| Users | Keep their password and authenticator app safe; are responsible for the API keys they add and for using each provider within that provider's terms; use DigiAsh only for lawful investigation. |
| Service providers | Cloudflare, the email provider and the threat-intelligence providers operate their services under their own terms and security programmes (see section 8). |

DigiAsh is operated by a single maintainer, so duties that a larger organisation would split between people (for example, development and production access) cannot be fully separated. This is compensated for by automated checks, multi-factor authentication on infrastructure accounts, and recorded change history.

## 5. Information classification

| Class | Examples | Handling |
|---|---|---|
| **Secret** | Users' provider API keys, MFA secrets, the master encryption key, the session-signing key, password-reset and email-verification tokens | Encrypted or hashed at rest; never logged, displayed after entry, committed to source control or sent to the browser in readable form. Infrastructure secrets live only in Cloudflare's secret store. |
| **Confidential** | Email addresses, password hashes, search history, contact-form messages | Stored in D1; accessible only to the account it belongs to (or the maintainer for operation and support); never shared except as described in the Privacy Policy. |
| **Internal** | Security logs, configuration, operational notes | Kept within the maintainer's accounts. |
| **Public** | The website content and public documentation | No restrictions. |

## 6. Risk management

Security risks are identified and assessed when features are designed and before releases, and the full application is reviewed periodically. The most recent assessment (November 2026, including tests against the live service) covered static and dynamic security testing, dependencies, the OWASP Top 10:2025, GDPR, these principles and the providers' terms; its findings were fixed or, where they need an account setting or a provider's action, scheduled with an owner. Each risk is treated by fixing it, reducing it with additional controls, or accepting it with a recorded reason.

## 7. Controls

### 7.1 Access control and authentication

- **Passwords:** at least 10 characters with a capital letter and a special character; checked against known breached passwords (Have I Been Pwned, using k-anonymity so the password never leaves DigiAsh); the last 3 passwords cannot be reused; accounts without MFA must change their password every 180 days.
- **Multi-factor authentication:** TOTP with any authenticator app, set up by scanning a QR code drawn in the browser (the secret is never sent to a third-party service) or by entering a setup key. Each code can be used only once. Ten one-time recovery codes, stored only as hashes, cover a lost device; a new set needs a current authenticator code. Setting up a new authenticator requires MFA to be turned off first, which requires the password.
- **Brute-force protection:** per-address rate limits on all sign-in endpoints; per-account lockouts after repeated wrong passwords or codes; a Cloudflare Turnstile bot check on signup, password-reset requests, the contact form and repeated failed logins.
- **Sessions:** signed session tokens in `HttpOnly`, `Secure`, `SameSite=Strict` cookies. The cookie is deleted when the browser closes, and the server ends a session after 30 minutes without activity and 12 hours after sign-in, whatever the browser keeps. Every session is checked against the account on each request, so all sessions end after a password reset or an MFA change, and users can sign out all other devices. Requests that change data are refused when they come from another site.
- **Account enumeration:** signup, login and password-reset responses do not reveal whether an email address has an account.
- **Third-party sign-in (Cloudflare):** only accounts created through Cloudflare can sign in this way; existing password accounts are never linked automatically.
- **Infrastructure access:** the Cloudflare, source-hosting and email-provider accounts used to run DigiAsh must be protected with strong unique passwords and multi-factor authentication.

### 7.2 Cryptography

- Users' API keys and MFA secrets are encrypted with AES-256-GCM using a master key held only in Cloudflare's secret store.
- Passwords are hashed with PBKDF2-SHA256 (100,000 iterations, the most Cloudflare Workers supports) and a unique random salt, then keyed with a secret pepper held only in Cloudflare's secret store, so a copy of the database alone can't be used to crack them.
- Password-reset and email-verification tokens are random 256-bit values, stored only as SHA-256 hashes, single-use and time-limited (30 minutes and 24 hours).
- All traffic is served over HTTPS with HTTP Strict Transport Security.
- Production secrets are generated specifically for production and are never reused from development.

### 7.3 Secure development

- Security requirements are part of each feature's design.
- Every change is type-checked and tested before it is committed, and an automated suite of unit, API and browser tests runs on every push.
- Output from providers and users is escaped before it is displayed, and links from providers are shown only if they use `http` or `https`.
- A strict Content Security Policy blocks inline scripts and limits pages to DigiAsh's own files (fonts are self-hosted), apart from Cloudflare's bot check. Security headers (HSTS, `X-Content-Type-Options`, `Referrer-Policy`, `Permissions-Policy`, framing protection) are sent on every response.
- Secrets are never committed to the repository. The repository history has been scanned for secrets. Dependabot checks the dependencies for updates and known vulnerabilities. *Planned:* automated secret scanning.
- Dependencies are kept minimal (one runtime dependency) and audited for known vulnerabilities before releases.

### 7.4 Third-party providers

DigiAsh sends the indicator being searched (an IP address, domain or file hash) to the threat-intelligence providers the user has added a key for, using the user's own key. Submissions to urlscan.io are made as *unlisted*, so they do not appear in its public feed. Each provider processes these requests under its own terms; DigiAsh is not affiliated with any of them. Supporting services are listed in section 8.

### 7.5 Logging and monitoring

- Errors are logged through Cloudflare's Workers Logs (kept for 3 days) for troubleshooting. Per-request logs are turned off, so the indicators users search aren't recorded there.
- Unexpected errors email the maintainer, at most once an hour, with only the request method, path and error. An external uptime monitor checks every 5 minutes that the service and its database respond.
- Logs must never contain passwords, API keys, MFA codes or secrets, session tokens, or reset and verification links. Links in emails are printed to the console only during local development.

### 7.6 Data retention

| Data | Retention |
|---|---|
| Account details | Until the account is deleted |
| Provider API keys | Until the user deletes them or their account |
| Search history | The most recent 50 searches per account |
| Search results | Up to 5 minutes in Cloudflare's cache, so a repeat search doesn't query the providers again; then deleted automatically |
| Failed sign-in records | Only as long as needed for the lockout window (minutes, up to a day for email limits) |
| Unconfirmed accounts | Deleted 7 days after signup if the email address is never confirmed |
| Reset and verification tokens | Unusable after use or expiry; deleted after 24 hours |
| Error logs | 3 days in Cloudflare Workers Logs, without search terms or account details |
| MFA recovery codes | Until used, replaced with a new set, or MFA is turned off |
| Contact-form messages | Until handled and removed by the maintainer, and with the account |

Users can download a copy of their data and delete their account, with all associated data, from the Security panel of the dashboard. Deleting an account requires the password and, where MFA is on, a second-factor code.

### 7.7 Backup and continuity

The database relies on Cloudflare D1's point-in-time recovery for the retention period Cloudflare provides on the current plan. The restore procedure is documented and was tested on a scratch database on 3 November 2026: a deleted row was recovered with D1 Time Travel. The service can be redeployed from the source repository and the documented production setup.

### 7.8 Vulnerability management

- Dependencies are checked for known vulnerabilities before each release and fixed promptly.
- Security reports are welcome through the contact page. Please do not test against other users' accounts or data, and give reasonable time for a fix before public disclosure. A `security.txt` file at `/.well-known/security.txt` gives the security contact.

## 8. Supporting services

| Service | Purpose | Data involved |
|---|---|---|
| Cloudflare | Hosting, database, key storage, DNS lookups, bot checks | All service data, under Cloudflare's security programme |
| Email provider (Resend) | Account emails (verification, password reset, notices) | Email address and message content |
| Have I Been Pwned (Pwned Passwords) | Breached-password check | The first 5 characters of a password hash only |
| ipwho.is, IANA and RDAP registries | Network and registration context for a searched indicator | The searched indicator |
| Threat-intelligence providers chosen by the user | The lookups themselves | The searched indicator and the user's own API key |

## 9. Incident management

If a security incident is suspected, the maintainer will:

1. **Identify** what happened and which data or accounts are affected.
2. **Contain** it, for example by rotating affected secrets, ending all sessions, or disabling the affected feature.
3. **Eradicate** the cause and **recover** the service, verifying the fix.
4. **Notify** affected users without undue delay when their data may be affected, and, where a personal data breach is likely to result in a risk to people's rights and freedoms, the relevant data protection supervisory authority within 72 hours of becoming aware of it, as the GDPR requires.
5. **Review** the incident afterwards and update controls and this policy.

## 10. Compliance

DigiAsh aims to comply with the EU General Data Protection Regulation (GDPR), including data minimisation, purpose limitation, security of processing and data subjects' rights; see the Privacy Policy. It also operates within the terms of each third-party service it uses.

## 11. Review

This policy is reviewed at least once a year and after any significant change to the service, its infrastructure or its risks.

| Version | Date | Change |
|---|---|---|
| 1.0 | October 2026 | First version |

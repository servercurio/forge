# Security Policy

## Reporting a Vulnerability

**Please do not report security vulnerabilities through public GitHub issues, discussions, or pull requests.**

This repository uses GitHub's [private vulnerability reporting](https://docs.github.com/en/code-security/security-advisories/guidance-on-reporting-and-writing-information-about-vulnerabilities/privately-reporting-a-security-vulnerability) to receive reports privately. To submit a report:

1. Open <https://github.com/servercurio/forge/security/advisories/new>.
2. Provide a clear description of the issue, the affected page or asset, and a minimal reproduction (commit hash, URL, sample content, etc.).
3. Suggest an impact assessment if you are able (CVSS vector or plain-language severity).

If you cannot use GitHub's reporting flow, email the maintainer at **nathan@theklickfamily.com** with the subject line `[SECURITY] forge`. Encrypted mail is welcome; ask for a current public key in your initial (unencrypted) message.

## What to Expect

- **Acknowledgement** within 3 business days of receipt.
- **Initial triage** (severity assessment, scope confirmation) within 7 business days.
- **Status updates** at least every 14 days while the report is open.
- **Coordinated disclosure** through a GitHub Security Advisory once a fix is available; reporters are credited unless they request otherwise.

We aim to publish fixes for confirmed vulnerabilities within 90 days of triage. More severe issues are prioritised; complex issues affecting external dependencies may take longer and will be communicated as such.

## Scope

This policy covers the source in this repository — documentation, design assets, and the website — and the site published from it.

In scope:

- Content-injection or cross-site-scripting issues introduced by the site's templates, shortcodes, or build pipeline.
- Misconfigurations in the repository's CI workflows that could compromise the published site or the repository itself.
- Leaked secrets or credentials committed to the repository.

Out of scope:

- Vulnerabilities in third-party dependencies (Hugo, themes, npm packages). Report those upstream; if exploitation requires this project to expose them in a non-default way, that part of the chain is in scope.
- Findings from automated scanners without a working proof of concept.
- Issues in the hosting/CDN platform itself rather than in this repository's content or configuration.

## Supported Versions

This repository publishes a rolling website from its `main` branch. Only the currently published site receives security fixes; there are no versioned releases to backport to.

## Safe Harbor

Good-faith security research conducted under this policy will not result in legal action from the maintainer. "Good faith" means: avoiding privacy violations, data destruction, service disruption, or access beyond what is necessary to demonstrate the issue, and giving us reasonable time to respond before public disclosure.

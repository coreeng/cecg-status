# 1. Use Upptime for the status page

Date: 2026-04-21

## Status

Accepted

## Context

PT-397 requires a public status page for core-platform applications as evidence for ISO 27001 / SOC 2 audits. The requirement has four load-bearing properties:

- **Public and HTTPS-served.**
- **Independent of the applications it monitors.** If every monitored application is down, the status page must still be reachable and continue to record their state.
- **Auditable.** Check frequency, incident history, human acknowledgement, and change provenance must be extractable without building a custom logging pipeline.
- **Notification-capable.** There must be a human-facing channel for incident open / close events.

The ticket asks us to evaluate open-source or low-overhead tools before building anything custom.

## Considered Options

- **Build custom in-house.** Maximum control, highest engineering cost, and requires independent hosting infrastructure to satisfy the independence requirement — moving the problem rather than solving it.
- **Self-hosted Uptime Kuma, Cachet, Statping, or Gatus.** Feature-rich, YAML-driven, and well-supported. Each requires a long-running server. Hosting that server on core-platform infrastructure would break independence; hosting it elsewhere reintroduces infrastructure we otherwise do not need to maintain.
- **SaaS status-page vendors (Statuspage.io, StatusGator, Better Uptime).** Satisfies independence by construction. Adds per-seat cost and a third-party vendor review. Audit trail accessible only through vendor export formats.
- **Upptime.** An open-source uptime monitor that runs entirely on GitHub: scheduled GitHub Actions probe endpoints, commit check history back to the repo, open GitHub Issues for incidents, and publish a static Svelte site to GitHub Pages. No server to run.

## Decision

We will use Upptime, operated out of this repository, `coreeng/cecg-status`.

Probes run on GitHub Actions cron. Check history, incident Issues, and the static site output all live in this repo. The published status page is served from GitHub Pages at `status.cecg.io`. Notifications are delivered to Slack via incoming webhook. Human acknowledgement is recorded as comments on the incident Issue.

The set of monitored services is authored by core-platform-dashboard and published to this repo as a generic services manifest — see ADR 0002.

## Consequences

- Good: no infrastructure to maintain. GitHub operates the scheduler, the runners, the static-site host, and the TLS certificate.
- Good: the independence requirement is satisfied structurally. No part of the monitoring system runs on infrastructure shared with the monitored applications.
- Good: the audit trail is a side effect of normal operation. Check frequency is visible in workflow-run history; incident timelines are visible on Issues; change provenance is visible in `git log` on `.upptimerc.yml`. No additional logging system is required.
- Good: no per-seat or per-check cost.
- Bad: resolution is capped at the Actions cron cadence. Practical minimum is five minutes. Sub-minute detection is not achievable with this architecture.
- Bad: the system's own availability depends on GitHub Actions and GitHub Pages. A GitHub incident degrades our status page. This is acceptable because GitHub publishes its own status page independently of ours.
- Bad: Upptime only supports URL-poll monitoring. PT-397 mentions metric-based monitoring as an option; this is not supported natively and is deliberately out of scope at this revision.
- Neutral: upstream Upptime is actively maintained. Breaking upstream changes would require an upgrade commitment. The template is pinned in `package.json` / workflow files and upgrades go through normal PR review.

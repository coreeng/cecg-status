# Upptime configuration for `cecg-status` — design

**Status:** Draft pending user review
**Date:** 2026-04-20
**Owner:** Jescard Tamer
**Tracks:** [PT-397 — Core Platform Status Page (for apps)](https://cecg-force.atlassian.net/browse/PT-397)
**Parent epic:** PT-278 — ISO 27001 requirements for PT

## 1. Context and motivation

PT-397 requires a public, ISO 27001 / SOC-compliant status page covering four core platform applications:

- `cecg.io`
- `coreplatform.io`
- `community.cecg.io`
- `osterley.cecg.io`

The status page must be independent from the applications it monitors — it cannot go down when those applications do — and it must be reachable over HTTPS. Evidence for auditors must include a monitoring trail, incident history, and a human notification path.

We are using [Upptime](https://upptime.js.org/), an open-source uptime monitor that runs entirely on GitHub Actions, commits check history back to the repo, opens GitHub Issues for incidents, and publishes a static status page to GitHub Pages. This model satisfies the independence requirement by design: no part of the monitoring system runs on the monitored infrastructure.

The repo `coreeng/cecg-status` has already been created from the Upptime template. This document specifies how we configure it.

## 2. Decisions

| Decision | Choice |
|---|---|
| GitHub authentication | GitHub App `cecg-status-bot` installed only on this repo |
| Public domain | `status.cecg.io` |
| Health check | `GET /livez` on each site, expect 2xx, every 5 minutes |
| Sites monitored | `cecg.io`, `coreplatform.io`, `community.cecg.io`, `osterley.cecg.io` |
| Notifications | Slack channel + GitHub Issue (audit trail) |
| Page name | "CECG Status" |
| Intro text | "Real-time status for CECG core platform applications." |
| Theme | `dark` |
| Logo | TBD — see §8 |
| Incident assignment | None — first responder picks it up |
| Rollout strategy | Two-phase staged rollout (§4, §5) |

## 3. Architecture

Nothing in this system runs on CECG-controlled infrastructure. All components are hosted by GitHub:

- **Repo `coreeng/cecg-status`** holds the configuration, accumulated check history, and the static site output on the `gh-pages` branch.
- **GitHub App `cecg-status-bot`** (installed only on this repo) provides the service identity that commits history and manages incident Issues. Its App ID and private key are stored as repo Actions secrets.
- **GitHub Actions workflows** (shipped by the Upptime template) run the scheduled probes, regenerate response-time graphs, build the static Svelte site, and push the build to `gh-pages`.
- **GitHub Pages** serves the `gh-pages` branch. Phase 1 serves from `coreeng.github.io/cecg-status`; Phase 2 cuts over to `status.cecg.io` with a GitHub-managed Let's Encrypt certificate.
- **Slack incoming webhook** (Phase 2 only) receives notifications on state transitions. The webhook URL is stored as a repo Actions secret.

### Independence guarantee

None of `cecg.io`, `coreplatform.io`, `community.cecg.io`, or `osterley.cecg.io` host any part of this system. If all four go down simultaneously, `status.cecg.io` remains reachable and continues to reflect their state.

### Audit trail

Every artifact auditors need is produced as a side-effect of normal operation:

- **Check frequency:** workflow-run history for `uptime.yml` — every run timestamped, attributed to the GitHub App identity.
- **Incident history:** the repo's Issues tab filtered by `label:status` — open/acknowledge/close timeline with author attribution.
- **Configuration-change provenance:** `git log` on `.upptimerc.yml` — each change goes through PR review.
- **Notification delivery:** workflow logs plus Slack channel archive.

## 4. Phase 1 — MVP on `github.io` default domain

Goal: working end-to-end on the default `coreeng.github.io/cecg-status` URL with no external dependencies (no Slack, no DNS).

### 4.1 GitHub App setup

Create a GitHub App named `cecg-status-bot` at the `coreeng` org level. Install only on `cecg-status`.

**Repository permissions (minimum set Upptime requires):**
- Contents: Read & Write
- Issues: Read & Write
- Pages: Read & Write
- Workflows: Read & Write
- Metadata: Read

**No org permissions, no user permissions.**

Generate the App ID and download a private key (`.pem`). Store both as repo Actions secrets:
- `GH_APP_ID`
- `GH_APP_PRIVATE_KEY`

Workflows mint a short-lived installation token on each run using `tibdex/github-app-token@v2`. No long-lived PAT is used.

### 4.2 `.upptimerc.yml`

Replace the Upptime template defaults with:

```yaml
owner: coreeng
repo: cecg-status

sites:
  - name: cecg.io
    url: https://cecg.io/livez
  - name: Core Platform
    url: https://coreplatform.io/livez
  - name: Community
    url: https://community.cecg.io/livez
  - name: Osterley
    url: https://osterley.cecg.io/livez

status-website:
  baseUrl: /cecg-status
  name: CECG Status
  introTitle: CECG Status
  introMessage: Real-time status for CECG core platform applications.
  theme: dark
  navbar:
    - title: Status
      href: /
    - title: GitHub
      href: https://github.com/coreeng/cecg-status
```

Deliberately absent in Phase 1: `cname`, `logoUrl`, and any `notifications` block.

### 4.3 Workflow token wiring

The Upptime template's workflows resolve their token from `secrets.GH_PAT` by default. Patch every workflow under `.github/workflows/` to instead mint an installation token from the GitHub App credentials. This is a small, consistent edit applied across the full set of workflow files.

### 4.4 GitHub Pages

Repo **Settings → Pages**:
- Source: **Deploy from branch**
- Branch: `gh-pages`, path `/` (root)
- Enforce HTTPS: on
- Custom domain: blank in Phase 1

### 4.5 Explicit non-goals for Phase 1

- No Slack integration (no webhook secret, no `notifications` block).
- No custom domain (no `cname` key, no DNS change).
- No logo or custom CSS.
- No incident auto-assignment.
- README stays as the Upptime template default; it will be rewritten in Phase 2 when branding is finalised.

### 4.6 Phase 1 exit criteria

Phase 1 is complete when all of the following hold for a continuous 48-hour window:

- `Uptime CI` runs every 5 minutes without failed runs (excluding intentional induced failures).
- `https://coreeng.github.io/cecg-status/` renders all four sites, each with up-to-date state and populated response-time sparklines.
- An induced outage (see §6.1) produces a GitHub Issue within one check cycle and auto-closes on recovery.

## 5. Phase 2 — Production posture

Goal: custom domain, live Slack alerting, final branding, incident runbook.

### 5.1 Entry gate

Phase 2 does not start until:

- Slack channel name and incoming webhook URL are provisioned (§8.1).
- Canonical CECG logo URL is identified and publicly reachable (§8.2).
- DNS owner for `cecg.io` is identified and the `status.cecg.io` record is approved (§8.3).

Each is tracked as a follow-up task under PT-397.

### 5.2 Slack integration

- Store the webhook URL as an Actions secret: `NOTIFICATION_SLACK_WEBHOOK_URL`.
- Add to `.upptimerc.yml`:
  ```yaml
  notifications:
    - type: slack
      channel: <confirmed-channel-name>
  ```
- Upptime's `uptime.yml` posts to the webhook on each up→down and down→up transition. The GitHub Issue record is unaffected — Slack is additive.

### 5.3 Custom domain cutover

1. Create DNS record with the domain owner: `status.cecg.io  CNAME  coreeng.github.io.` (TTL 300).
2. In `.upptimerc.yml`, replace `baseUrl: /cecg-status` with `cname: status.cecg.io`. The `static-site.yml` workflow will write a matching `CNAME` file into the `gh-pages` build on the next deploy.
3. Repo **Settings → Pages → Custom domain:** `status.cecg.io`. Wait for GitHub's DNS check to pass, then enable **Enforce HTTPS**. GitHub provisions a Let's Encrypt certificate automatically.
4. Verify `https://status.cecg.io` serves the same content as the Phase 1 `github.io` URL.

### 5.4 Branding

- Add `logoUrl: <cecg-logo-url>` to the `status-website` block.
- If pixel-accurate CECG colours are required, add `assets/brand.css` with CSS variable overrides for the dark theme and reference via `themeUrl: /assets/brand.css`. This is a stretch goal within Phase 2, not a blocker — the default dark theme is acceptable for go-live.

### 5.5 Incident runbook

Commit a new file at `docs/incident-runbook.md` describing:

- What triggers an incident Issue.
- How on-call acknowledges and takes ownership (by commenting on the Issue).
- Response-time targets: acknowledgement within 15 minutes, resolution target within 4 hours (team to confirm / adjust).
- Communication template for external-facing updates posted as Issue comments (rendered as incident updates on the public page).
- Post-incident review expectation: link from the Issue to any follow-up RCA document.

This runbook is a deliberate PT-397 deliverable — auditors expect it alongside the page itself.

### 5.6 README

Rewrite `README.md` to describe:

- What the repo is and what it monitors.
- How to read the status page and the Issues list.
- How to add a new monitored site (small PR edit to `.upptimerc.yml`).
- Contact / ownership.

### 5.7 Phase 2 exit criteria

- `https://status.cecg.io` loads over HTTPS with a valid cert.
- Induced outage (§6.2) produces a Slack message in the agreed channel in addition to the Issue.
- Slack webhook failure mode verified: a broken webhook does not prevent the Issue from being opened.
- Logo renders, dark theme applied.
- `README.md` and `docs/incident-runbook.md` committed, reviewed on PR.

## 6. Operational flow

This section describes the steady state after Phase 2 is complete. During Phase 1, the same flows apply with all Slack steps omitted — the GitHub Issue and workflow-runs history still form a complete audit trail on their own.

### 6.1 Steady state (every 5 minutes, healthy)

1. GitHub cron fires `uptime.yml`.
2. Workflow mints a short-lived installation token from the GitHub App.
3. For each site: `curl https://<host>/livez` and capture status code + response time.
4. Commit updates to `history/<slug>.yml` and `api/<slug>/*.json` on the default branch.
5. That commit triggers `static-site.yml` → rebuilds the Svelte site → force-pushes `gh-pages`.
6. GitHub Pages serves the new build within ~30 seconds.
7. No Issue opened, no Slack message sent.

### 6.2 Incident open

1. A `/livez` check returns non-2xx or times out.
2. Workflow opens a GitHub Issue titled `🛑 <Site> is down`, labelled `status`, `down`, with the response code, timestamp, and a link to the workflow run.
3. Slack webhook posts to the agreed channel with a link to the Issue.
4. First responder acknowledges by commenting on the Issue. This comment is the audit-trail entry for "human acknowledged at T+X".
5. The Issue body and subsequent comments render as live incident updates under the affected site on `status.cecg.io`.

### 6.3 Incident close

1. The next 5-minute check for the same site returns 2xx.
2. Workflow posts a closing comment on the open Issue and closes it.
3. Slack receives a recovery message.
4. The status page flips the site row back to green. The closed Issue appears in the site's "past incidents" list with its full timeline preserved.

### 6.4 Evidence for an auditor

From the `coreeng/cecg-status` repo:

- **Check frequency:** the `uptime.yml` workflow-runs list.
- **Incident history:** Issues filtered by `label:status`; each Issue carries create/acknowledge/close timestamps and author identity.
- **Response latency:** Issue metadata (created → first human comment) and Slack archive.
- **Configuration-change provenance:** `git log` on `.upptimerc.yml`.

No external logging or monitoring system is required to produce this package.

## 7. Testing and validation

Because this is a configuration change rather than application code, "testing" means explicit verification steps executed at each phase gate. The steps below become evidence for PT-397 closure.

### 7.1 Phase 1 validation

1. **Token resolution.** Manually trigger `uptime.yml` via `workflow_dispatch`. Workflow completes successfully — proves `GH_APP_ID` and `GH_APP_PRIVATE_KEY` are correct.
2. **First scheduled run.** Confirm `history/*.yml` receive commits attributed to `cecg-status-bot[bot]`.
3. **Static site deploys.** Visit `https://coreeng.github.io/cecg-status/`. All four sites appear. Sparklines populate after two or more runs.
4. **Induced outage.** Temporarily repoint one check at a URL guaranteed to fail (e.g. `https://cecg.io/livez-does-not-exist`). Within ~5 minutes: a GitHub Issue opens, the public page flips that site to red. Revert the config; confirm the next run auto-closes the Issue and the row returns to green.
5. **Stability soak.** Leave Phase 1 running for 48 hours. No failed workflow runs (other than the induced test), no false-positive Issues.

Phase 2 does not start until the 48-hour soak passes.

### 7.2 Phase 2 validation

1. **DNS propagation and TLS.** After cutover: GitHub's domain check passes and `https://status.cecg.io` serves the page with a Let's Encrypt cert issued to the new hostname.
2. **HTTP → HTTPS redirect.** `curl -I http://status.cecg.io` returns a 301 or 308 to the HTTPS variant.
3. **Slack end-to-end.** Repeat the induced outage from §7.1 step 4. Verify the message arrives in the agreed channel.
4. **Slack failure mode.** Temporarily set `NOTIFICATION_SLACK_WEBHOOK_URL` to a 404 URL. Run an induced outage. The GitHub Issue still opens; Slack step fails visibly in the workflow log but does not block incident recording. Restore the secret afterwards.
5. **Branding.** Logo renders, dark theme applied, name and intro text match the spec.
6. **Runbook review.** `docs/incident-runbook.md` reviewed by at least one other team member via PR.

### 7.3 PT-397 closure checklist

- [ ] Status page reachable at `https://status.cecg.io` over HTTPS.
- [ ] All four sites monitored every 5 minutes against `/livez`.
- [ ] Status page is independent of the monitored apps.
- [ ] Incident audit trail exists (Issues + workflow-runs history).
- [ ] Human notification path wired (Slack) and tested.
- [ ] Incident runbook committed.

## 8. Open dependencies (TBDs)

Each is tracked as a follow-up task under PT-397. Phase 2 is gated on all three resolving.

### 8.1 Slack channel and webhook

- Confirm which channel to use (reuse existing alerts channel vs dedicated `#status-page-alerts`).
- Identify who provisions the incoming webhook (self-service by Jescard vs. Slack admin request).

### 8.2 Logo asset

- Identify canonical CECG logo SVG at a public URL, or agree on a hosted copy location within the repo.

### 8.3 DNS owner for `cecg.io`

- Identify DNS provider (Cloudflare, Route53, other) and the approver for adding `status.cecg.io`.

## 9. Out of scope

These are deliberate exclusions, not oversights:

- **Load testing the status page itself.** It is served by GitHub Pages; capacity is GitHub's responsibility.
- **Correctness testing of `/livez` endpoints.** If an application's `/livez` returns 200 while the app is broken, that is a separate issue for the owning team to address in their own health endpoint.
- **Escalation beyond Slack.** If Slack is unavailable and the webhook fails silently, that is an accepted risk at this posture. PagerDuty or similar escalation tooling is out of scope for PT-397; it can be layered in later if the on-call maturity calls for it.
- **Authenticated health checks.** All four sites' `/livez` endpoints are assumed to be publicly reachable without authentication. If any require auth in future, we can add `headers` entries to the site config.

## 10. References

- PT-397: <https://cecg-force.atlassian.net/browse/PT-397>
- Upstream Upptime: <https://github.com/upptime/upptime>
- Upptime configuration docs: <https://upptime.js.org/docs/configuration>
- Repo: <https://github.com/coreeng/cecg-status>

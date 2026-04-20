# CECG Status Page — Incident Runbook

How to respond to incidents detected by the public status page at
[status.cecg.io](https://status.cecg.io) (during Phase 1, served from
[coreeng.github.io/cecg-status](https://coreeng.github.io/cecg-status/)).
The page monitors the four core platform sites defined in
[`.upptimerc.yml`](../.upptimerc.yml). Incidents are recorded as GitHub
Issues in [`coreeng/cecg-status`](https://github.com/coreeng/cecg-status)
and mirrored to the agreed Slack alerts channel.

This document exists as part of the ISO 27001 / SOC 2 control evidence
for incident response and is the primary reference for on-call responders.

## 1. How incidents are detected

Every 5 minutes, the `Uptime CI` workflow issues an HTTP `GET` to `/livez`
on each monitored site:

- `cecg.io`
- `coreplatform.io`
- `community.cecg.io`
- `osterley.cecg.io`

A probe is considered failed if the response is any non-2xx status code or
the request times out. On a failed probe, the workflow automatically:

1. **Opens a GitHub Issue** in this repo, titled `🛑 <Site> is down`, labelled
   `status` and `down`. The Issue body contains the failing response code,
   a timestamp, and a link to the `Uptime CI` workflow run.
2. **Posts to Slack** via the configured incoming webhook, with a short
   summary and a link to the new Issue.
3. **Flips the row** for the affected site to red on the public page within
   the next build cycle (typically 1–2 minutes after the Issue opens).

The Issue and the Slack message are the two audit-trail entry points. If
Slack is unavailable or the webhook fails, the Issue is still created.

## 2. Acknowledging an incident

The first responder to see the incident **must acknowledge it by commenting
on the Issue** with a short note. For example:

> Acknowledged by @jescard at 14:03 UTC — investigating.

This comment's timestamp is the audit-trail record of "a human saw this and
took ownership at T+X." Without an acknowledgement comment, auditors cannot
demonstrate that the incident was responded to. **Do not rely on Slack
reactions or DMs for acknowledgement** — those do not produce a durable,
linkable record.

## 3. Response targets

These are the team's starting commitments. They should be reviewed with the
delivery manager after the first quarter of operation and revised if needed.

| Target | Value |
| --- | --- |
| Acknowledgement (first comment on Issue) | within **15 minutes** of Issue open |
| Resolution (Issue auto-closes on recovery) | within **4 hours** of open |
| Customer-facing status update cadence | at minimum, every **30 minutes** while red |

If an incident is expected to exceed the resolution target, post a comment
on the Issue stating the revised estimate and reason — see §4.

## 4. External communication

Incident Issue comments render as live incident updates on the public page.
Anything you post becomes publicly visible. Use this template for updates:

> **[HH:MM UTC]** We are investigating reports of elevated errors affecting
> `<site>`. Users may experience intermittent failures. Further updates to
> follow.
>
> **[HH:MM UTC]** We have identified the cause as `<brief cause>` and are
> deploying a fix. ETA to resolution: `<estimate>`.
>
> **[HH:MM UTC]** The issue is resolved. A post-incident review will be
> published within 5 business days.

Guidance:

- **Write for external customers**, not for the team. Assume the reader
  does not know our internal services.
- **Avoid blame, speculation, or premature root-cause claims.** Stick to
  observable impact and the current state of the investigation.
- **Do not paste stack traces, internal URLs, or credentials.** These
  comments are permanent and indexable.
- If the incident turns out to be a false alarm (e.g. our probe endpoint
  broke while the service is healthy), post a single clarifying update and
  close the Issue manually.

## 5. Recovery

When the next `/livez` probe returns 2xx, the `Uptime CI` workflow
automatically:

1. Posts a closing comment on the open Issue
2. Closes the Issue
3. Sends a recovery message to Slack
4. Flips the site's row back to green on the public page

**No manual action is required to close a recovered incident.** If you
close an Issue manually (e.g. because it was a false alarm), add a comment
explaining why before closing.

## 6. Post-incident review

For any incident that caused **customer-visible impact** (not every probe
failure does), a post-incident review (PIR) is expected within 5 business
days. The PIR covers:

- Timeline of detection, response, and resolution
- Root cause(s)
- Contributing factors
- Actions taken during the incident
- Follow-up actions and owners

PIRs live alongside other core platform engineering documents. Link from
the original Issue to the PIR document once published. _(Exact location of
PIRs to be confirmed with the delivery manager — placeholder until team
convention is set.)_

## 7. Escalation

Current posture (Phase 2 launch):

- **Primary channel:** GitHub Issue + Slack message to the core platform
  alerts channel
- **Secondary:** responders are expected to monitor their repo Issue
  subscriptions as a backstop for Slack outages

PagerDuty / Opsgenie / phone-tree escalation is **out of scope at launch**
and is a deliberate accepted risk — see the design spec at
`docs/superpowers/specs/2026-04-20-upptime-cecg-status-design.md`. Revisit
if repeated incidents occur outside working hours or if response targets
are routinely missed.

## 8. Common operational tasks

### Add a new monitored site

Edit [`.upptimerc.yml`](../.upptimerc.yml) under `sites:`:

```yaml
  - name: <Display Name>
    url: https://<host>/livez
```

Open a PR. Once merged, the next `Setup CI` run picks it up and the site
appears on the public page within a few minutes.

### Remove a monitored site

Delete its entry from `.upptimerc.yml`. The historical data (`history/`,
`api/`, `graphs/` folders for that site) can be deleted in the same PR or
kept as archived evidence.

### Manually trigger a probe

Go to the [Actions tab](https://github.com/coreeng/cecg-status/actions/workflows/uptime.yml)
and click **Run workflow** on `Uptime CI`. Useful when you've just made a
fix and want to confirm recovery without waiting the full 5 minutes.

### Pause monitoring for planned maintenance

If a site is going into a known maintenance window and you want to suppress
false incident Issues, either:

1. **Temporarily comment out the site** in `.upptimerc.yml` for the
   duration, then restore it afterwards; or
2. **Accept the Issue and close it manually** with a comment explaining
   the maintenance, then re-close on recovery.

Option 1 is cleaner for longer windows; option 2 for short, ad-hoc cases.

## 9. Evidence package for auditors

When asked to produce evidence of incident response for ISO 27001 / SOC,
collect:

- Full list of `Uptime CI` runs for the audit period (via the
  [Actions UI](https://github.com/coreeng/cecg-status/actions/workflows/uptime.yml))
  demonstrating the 5-minute check cadence
- Full list of `status`-labelled Issues (open and closed) for the period
- Any post-incident reviews linked from the Issues
- This runbook itself as the procedural document
- The design specification in `docs/superpowers/specs/`

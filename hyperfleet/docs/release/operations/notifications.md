---
Status: Active
Owner: HyperFleet Team
Last Updated: 2026-10-01
---

# Notifications and Status Signals

> **Audience:** HyperFleet engineers wiring or troubleshooting build and release notifications, or trying to find out where to watch a release land.

HyperFleet build and release pipelines emit status signals to several places.
This page covers what they are, what posts there, and how to change or rotate
each one.

---

## Where to watch a release

| Surface | What lands there | Driven by |
|---------|-----------------|-----------|
| **Slack `#hyperfleet-e2e-status`** | Per-component release success/failure messages from `rh-push-to-external-registry` | `data.slack` in the RPA |
| **Slack build failure alerts** | Failed on-push, tag, and chart PipelineRuns | `slack-webhook-notification` final task in each component repository |
| **Konflux UI** | Pipeline runs, Snapshots, Releases, EC verdicts, scan results | Konflux platform |
| **Pyxis / Red Hat Container Catalog** | Image entry + metadata + CVE tracking | `create-pyxis-image` task in the release pipeline |
| **GitHub commit status** | PaC reports build pass/fail back to the commit | PaC controller |
| **Prow dashboard** | E2E results (nightly + RC) | Prow / ci-operator |

---

## Slack: `#hyperfleet-e2e-status`

The release pipeline posts to `#hyperfleet-e2e-status` on Red Hat Slack on each release run. The notification is part of the `rh-push-to-external-registry` pipeline's `finally` block — even if the push fails, the message goes out. If the message itself fails to send, the release still records as complete (the notification is best-effort).

What a successful message contains:

- Release CR name
- Application (`hyperfleet`)
- Snapshot
- Component images and the tags they were pushed with
- Link back to the Konflux UI

What a failed message contains:

- Same as success, plus the failed task name and a UI link to its logs

### Configuration

The RPA references two values in `data.slack`:

```yaml
data:
  slack:
    slack-notification-secret: hyperfleet-slack-webhook-notification-secret
    slack-webhook-notification-secret-keyname: webhook-url
```

The secret lives in the `rhtap-releng-tenant` namespace (the same namespace as the RPA). The HyperFleet team does not have direct write access there — RelEng manages it. The secret name and keyname are referenced by the RPA; the secret's value is an incoming-webhook URL for the channel.

File: `konflux-release-data/config/kflux-prd-rh02.0fk9.p1/service/ReleasePlanAdmission/hyperfleet/hyperfleet.yaml`.

### Adding a second channel

Konflux supports one webhook per RPA. To post to a second channel:

1. Create a second incoming webhook in Slack for the new channel.
2. Coordinate with RelEng to create a second secret in `rhtap-releng-tenant`.
3. The Konflux platform doesn't natively fan out — options are (a) make the webhook a Slack workflow that re-broadcasts, or (b) configure a Slack channel mirror. Option (a) is simpler.

If you need different content per channel, that's a custom finally-task — out of scope for `rh-push-to-external-registry`.

### Rotating the Slack webhook

1. Create a new incoming webhook in Slack for `#hyperfleet-e2e-status`.
2. Open an MR or ticket with RelEng asking them to update the `webhook-url` key on `hyperfleet-slack-webhook-notification-secret` in `rhtap-releng-tenant` on `kflux-prd-rh02`.
3. If the same webhook URL is also used for build alerts, coordinate the build-tenant Secret update and validate both consumers before revoking the old URL. See [Build failure alerts](#build-failure-alerts).
4. Revoke the old webhook in Slack after validation.

No `konflux-release-data` change is needed — the RPA references the secret by name, not the URL.

### Webhook missing / silent

If `#hyperfleet-e2e-status` stops getting messages:

1. Confirm a release actually fired: Konflux UI → Releases.
2. Open the release's managed PipelineRun → look for the `slack-notification` task in the `finally` block.
3. Common failures: secret rotated incorrectly, webhook URL revoked, Slack rate-limited. All show up in the task logs.

---

## Build failure alerts

The API, Sentinel, Adapter, Operator, and Applier repositories each define four
Konflux PipelineRuns with a `slack-webhook-notification` task in
`spec.pipelineSpec.finally`. The task runs only when `$(tasks.status)` is
`Failed`. Its message names the repository, pipeline, and commit, with a labeled
link to the failed run. Successful runs skip the task. It cannot alert on a
Pipelines as Code rejection or another failure that prevents a PipelineRun
from starting.

The message uses [Slack app link formatting](https://docs.slack.dev/messaging/formatting-message-text/)
such as `<URL|Open in Konflux>`. Tekton resolves the `$(...)` values before the
task posts the message. Pasting the YAML template into Slack's message editor
does not test the webhook output: the URLs still contain Tekton variables, and
the editor has [different link markup](https://slack.com/help/articles/360039953113-Format-your-messages-in-Slack-with-markup).

The build task reads `hyperfleet-slack-webhook-url` from Secret
`hyperfleet-slack-webhook-notification-secret` in the `hyperfleet-tenant`
namespace on `kflux-prd-rh02`. HyperFleet manages this namespace and Secret.
The release pipeline uses a separate Secret in `rhtap-releng-tenant`, managed
by RelEng. The webhook URL may be shared; check before rotating either Secret.
Do not put the URL in Git, a ticket, or task logs.

Each repository documents its PipelineRuns and webhook rotation:

- [API](https://github.com/openshift-hyperfleet/hyperfleet-api/blob/main/.tekton/README.md)
- [Sentinel](https://github.com/openshift-hyperfleet/hyperfleet-sentinel/blob/main/.tekton/README.md)
- [Adapter](https://github.com/openshift-hyperfleet/hyperfleet-adapter/blob/main/.tekton/README.md)
- [Operator](https://github.com/openshift-hyperfleet/hyperfleet-operator/blob/main/.tekton/README.md)
- [Applier](https://github.com/openshift-hyperfleet/hyperfleet-applier/blob/main/.tekton/README.md)

### Missing build alert

1. Open the failed build PipelineRun in the Konflux UI and inspect its
   `slack-webhook-notification` final task and logs. A missing Secret or key,
   task resolution failure, revoked URL, or Slack delivery error appears there.
2. Confirm the task can mount the Secret in `hyperfleet-tenant`; the release
   namespace Secret cannot be mounted across namespaces.
3. If no PipelineRun exists, inspect the commit's Pipelines as Code check and
   trigger rules. The `finally` task cannot cover a run that never started.

To rotate the build webhook, obtain a replacement for the approved channel and
update the build-tenant Secret through HyperFleet's Secret management process.
Update its configuration source if one is used. If the URL is shared with
releases, coordinate the release Secret update with RelEng. With authorization
for a controlled run and Slack post, verify a failed build alert arrives within a few
minutes with the correct fields and link, a successful build emits no failure
alert, and a release notification still arrives when the URL is shared. Only
then revoke the old webhook.

---

## Pyxis

Pyxis is Red Hat's container metadata catalog. The release pipeline registers each image so that:

- The image appears in the Red Hat Container Catalog.
- Continuous CVE scanning runs against it.
- RPM-level metadata is tracked (where applicable).

Configured via `data.pyxis` in the RPA (the secret itself is shared infrastructure managed by RelEng). To confirm a freshly-released image is in Pyxis, search by the SHA or repository name in the Red Hat Container Catalog UI — there's a propagation delay of a few minutes.

If `create-pyxis-image` fails in the release run, the image is still in Quay but missing from Pyxis. Re-running the release usually fixes it; persistent failures are a RelEng ticket.

---

## GitHub commit status (PaC)

PaC reports build pass/fail back to the GitHub commit. You'll see:

- A green check / red X next to the commit on the component repo.
- A "Konflux / hyperfleet-<svc>-on-push" or `…-on-tag` status line.
- Click → opens the PipelineRun in the Konflux UI.

This is the fastest way to confirm "did my push trigger a build" without opening the Konflux UI. If the status line is missing, the PaC trigger didn't fire — see [Debugging: I pushed a tag and no PipelineRun started](./debugging.md#i-pushed-a-tag-and-no-pipelinerun-started).

---

## Prow

E2E results land on the Prow dashboard, not in Slack (by default). Nightly E2E results are accessible from the periodic job; RC E2E results from the Gangway-triggered job. The `trigger-rc-e2e.sh` script prints a Prow job URL on submission — open that to follow the run.

For details on configuring or extending Prow notifications, see [Add Hyperfleet E2E CI Job in Prow](../test-release/add-hyperfleet-e2e-ci-job-in-prow.md).

---

## Konflux UI

- URL: <https://konflux-ui.apps.kflux-prd-rh02.0fk9.p1.openshiftapps.com/>
- The cluster is `kflux-prd-rh02`. Bookmark the URL — it's the single pane for everything that runs on our tenant.
- Sign-in is Red Hat SSO; you'll need access to the `hyperfleet-tenant` namespace via the team Rover group.

---

## Related

- [Configuration Map](./configuration-map.md) — exact file paths for the RPA and Slack config
- [Debugging](./debugging.md) — what to do when a release fails silently
- [Support](./support.md) — escalation contacts

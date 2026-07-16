# Filebeat Inode-Reuse Case — Follow-up Correspondence

Reference material for support Case **#[CASE NUMBER]** (Elastic engineer: Itzel).
Root cause under investigation: **inode reuse** causing silent log-ingestion gaps from Filebeat in Kubernetes.

---

## Context / summary of the case

Filebeat in our Kubernetes environment intermittently stopped shipping logs for certain pods. Support (Itzel) traced it to **inode reuse** — Filebeat's registry keys file state on inode + device ID, so when the OS reuses an inode from a deleted container log, Filebeat treats the "new" file as already-read and skips it, producing silent ingestion gaps. The recommended remediation points toward registry/rotation tuning and the `filestream` input with fingerprint-based file identity.

### Key technical points

- The definitive cure for inode reuse is migrating from the `log` input to `filestream` with `file_identity: fingerprint`, which identifies files by a content hash (SHA256 of the first ~1KB) instead of inode + device.
- `fingerprint` became the **default** file identity in Filebeat **9.0**. In **8.x** it must be configured explicitly.
- Switching from `native`/`path` to `fingerprint` **auto-migrates** existing offsets on startup (no full re-read), whereas any other switch causes re-ingestion/duplication.
- In the standard Filebeat DaemonSet, `path.data` (the registry) is mounted as a **hostPath on the node** precisely so it survives pod restarts — which means a plain **pod restart very likely will NOT clear the state**.

### What changed vs. the original draft

- Added questions about the **permanent fix** (filestream + fingerprint) and the **running version**, which were missing.
- Sharpened the pod-vs-node question to confirm the hostPath/registry behavior directly instead of leaving it open-ended.
- Reframed "manual intervention from Elastic" into a request for **interim mitigations/runbook** in the support reply, and moved the **escalation/push** ask into the account-team email, where it belongs.

---

## 1. Rewritten reply to Elastic Support (Itzel)

> Fill in placeholders and drop the "tell us which version/input" sub-parts if you already know them.

```
Hi Itzel,

Thank you for the detailed analysis on this case. Before we complete the
recommended changes, we'd like to lock down our understanding of a few points
so we can respond quickly if inode reuse recurs in production.

REGISTRY / METADATA STORAGE

1. Where exactly does Filebeat persist file state (the registry) in our
   deployment? Our understanding is that state is keyed on inode + device ID and
   written under path.data/registry. In our Kubernetes DaemonSet we believe
   path.data is mounted as a hostPath on the node so the registry survives pod
   restarts. Can you confirm whether that's the case here, and therefore whether
   the metadata lives on the node rather than in ephemeral pod storage?

2. Following from that: if inode reuse happens again, will restarting the
   affected pod clear the state and let logging resume? Our assumption is that if
   the registry is on a node-level hostPath, a plain pod restart will NOT clear
   it. Please confirm, and tell us what the correct recovery action actually is.

CLEARING STATE FOR SPECIFIC PODS/CONTAINERS

3. Besides restarting the pod, what are the supported ways to clear or reset the
   registry state for only the affected pod(s)/container(s), without forcing a
   full re-read (and duplication) of every other file? For example, is there a
   safe way to prune specific entries, or specific settings (clean_removed,
   clean_inactive, close_removed) we should rely on?

EVIDENCE / PROVING THE ROOT CAUSE

4. Is inode information recorded in the Filebeat logs, and if so at what log
   level and with which selectors? We currently have no empirical facts
   confirming inode reuse and would like to capture proof next time. Please tell
   us exactly what to enable (e.g. logging.level: debug, specific -d selectors)
   and what log signatures to look for (offset resets, truncation-detected
   messages, the state-id showing inode+device, etc.) so we can substantiate it
   definitively.

INTERIM STABILITY

5. While we roll out the recommended changes — which, in our change-controlled
   federal environment, takes time to test and deploy — what interim mitigations
   do you recommend to keep logging stable? Is there a config change or
   operational runbook we can apply now to reduce the likelihood or blast radius
   of a recurrence?

ADDITIONAL QUESTIONS

6. Permanent fix (filestream + fingerprint): Our understanding is that the
   definitive fix for inode reuse is the filestream input with
   file_identity: fingerprint (content-based identity rather than inode +
   device). Can you confirm (a) whether we're currently on the log input or
   filestream, (b) whether fingerprint is the recommended target for our
   environment, and (c) the exact migration path, including whether existing
   offsets auto-migrate without duplication?

7. Version specifics: Which Filebeat / Elastic Agent version are we running, and
   does fingerprint apply by default on that version (we understand it's default
   from 9.0) or does it need explicit configuration? Please give us the
   version-specific config.

8. Fingerprint edge cases: Many of our container logs may share identical opening
   bytes (e.g. common JSON prefixes). Should we tune the fingerprint
   offset/length so distinct files don't hash to the same identity, and what do
   you recommend?

9. Node-level log rotation: How does kubelet log rotation (containerLogMaxSize /
   containerLogMaxFiles) interact with this? Faster rotation increases inode
   churn — do you recommend any change there?

10. Proactive detection: Can you recommend a way to detect a recurrence early —
    e.g. an alert on stalled harvesters, registry growth, or a per-container
    ingestion gap — so we catch it before it impacts us?

11. Data-loss assessment: For the window when this occurred, can we determine
    what was missed, and is there any supported way to backfill the affected logs?

Thanks,
Ola
```

---

## 2. Email to the Elastic account team

Two variants depending on how you want to pitch it. Fill in the case number, account-team contact, and version placeholders.

### Variant A — Direct escalation

**Subject:** `Request to escalate — production Filebeat logging instability (Case #[CASE NUMBER])`

```
Hi [Account team name],

I want to make you aware of a production issue we're actively working with
Elastic Support on, and ask for your help pushing for prioritized engagement
from Elastic's side.

The situation: In our Kubernetes environment, Filebeat has intermittently
stopped shipping logs for certain pods. Working with Support (Case
#[CASE NUMBER], engineer Itzel), we've traced the root cause to inode reuse —
Filebeat's file-state registry keys on inode + device ID, and in a high-churn
containerized environment reused inodes cause harvesters to skip files,
producing silent gaps in log ingestion.

Why it matters: This logging underpins our monitoring, security, and compliance
visibility in a FedRAMP/FISMA federal environment, so any silent gap in log
delivery is a real operational and compliance concern for us.

Where we are: Support has recommended remediation (migrating to the filestream
input with fingerprint-based file identity, plus registry and rotation tuning).
We agree with the direction, but rolling this out in a change-controlled federal
environment requires testing and staged deployment — so it will take time,
during which we remain exposed to recurrence.

What we're asking: Could you help push for prioritized support engagement on our
behalf? Specifically:
- Escalation and prioritization of Case #[CASE NUMBER];
- Access to a Filebeat/Beats SME, ideally a short working session, to validate
  our target configuration for our exact version and deployment and to define an
  interim mitigation/runbook;
- Confirmation of best-practice guidance for inode-reuse resilience in
  Kubernetes at our scale.

Happy to set up a call with our team at your convenience. Thanks for any help you
can lend here.

Best,
Ola Olajide
Site Reliability & Observability Engineer, HedgeServ
[phone / email]
```

### Variant B — Relationship-led

**Subject:** `Heads-up + a favor on an active support case (Case #[CASE NUMBER])`

```
Hi [Account team name],

Hope you're doing well. I wanted to loop you in on something we're working
through with Elastic Support, and see whether you can help us get a bit more
momentum on it.

We've been troubleshooting intermittent log-shipping outages from Filebeat in
our Kubernetes environment. With Support (Case #[CASE NUMBER], engineer Itzel —
who has been great), we've narrowed the root cause to inode reuse: Filebeat
identifies files by inode + device ID, and in our high-churn containerized setup
reused inodes cause it to skip files, leaving silent gaps in our logs.

Given that this logging feeds our monitoring and compliance visibility in a
FedRAMP/FISMA environment, those gaps are something we need to close with
confidence. Support has pointed us toward the right remediation (moving to the
filestream input with fingerprint file identity, plus some tuning), and we're on
board — but validating and rolling that out under federal change control takes
time, and we'd like to shorten the exposure window.

That's where I'm hoping you can help. If you're able to:
- nudge Case #[CASE NUMBER] up the priority list,
- connect us with a Beats SME for a short working session to sanity-check our
  target config (version [VERSION], DaemonSet deployment) and agree an interim
  runbook,
- and confirm best-practice guidance for inode-reuse resilience in Kubernetes at
  our scale,

...it would make a real difference for us. Glad to jump on a call whenever works
for your team.

Thanks as always,
Ola Olajide
Site Reliability & Observability Engineer, HedgeServ
[phone / email]
```

---

## Before sending — checklist

- [ ] Fill in `[CASE NUMBER]`, `[Account team name]`, `[VERSION]`, and contact details.
- [ ] Confirm whether you're on the legacy `log` input or `filestream` — this determines which of questions 6/7 you can answer yourself.
- [ ] If on 9.x, `fingerprint` is already the default; reframe question 6 accordingly.
- [ ] The support reply intentionally states assumptions (hostPath on node, pod restart won't help) to force a confirm/correct answer rather than an open-ended reply.

# Chapter 10: Conclusion & Outcomes

## What was built

| Chapter | Artifact |
|---|---|
| 3 | Organization, credentials, inventory (`ocpvirt_vms` / `vmware_vms`), project, Execution Environments |
| 4 | `Backup VM - OpenShift Virtualization` and `Backup VM - VMware` job templates |
| 5 | `Patch & Health-Check VM` job template (platform-agnostic) |
| 6 | `VM Patch Cycle` workflow per platform, plus `Restore VM - *` job templates |
| 7–8 | EDA rulebook activation that auto-restores VMs based on delayed monitoring alerts |
| 9 | Consolidation into one workflow, three job templates, one Execution Environment, driven by `vm_platform` |

## Outcomes against the original use case

Going back to [Chapter 0](00-the-use-case.md), the four-step manual process
— **backup, patch, health-check, restore-if-needed** — is now:

- **Automated end-to-end** as a single workflow, launchable on demand or on
  a schedule, for either virtualization platform.
- **Self-correcting within the run** — a failed health check triggers an
  automatic restore to the exact snapshot taken for that run, via the
  `snapshot_name` artifact.
- **Self-correcting *after* the run** — Event-Driven Ansible watches for
  delayed failures and triggers the same restore job templates
  independently, using the most recent pre-patch snapshot.
- **Consistent across platforms** — the same workflow shape, the same job
  template names, and the same EDA rule structure apply whether a VM lives
  on OpenShift Virtualization or VMware. The platform difference is
  isolated to two small roles.

## Lessons from how this was built

- **Separate "what," "where," and "who/when."** Automation Controller
  (who/when, RBAC, scheduling) stayed constant throughout; only the
  Execution Environment (where) and the roles (what) needed
  platform-specific content.
- **Artifacts (`set_stats`) are the glue between workflow steps.** Passing
  `snapshot_name` from backup to restore avoided any hardcoded or
  out-of-band state.
- **A `fail` task is a feature, not just an error.** The health-check
  playbook deliberately fails the job when a VM is unhealthy — that failure
  *is* the signal the workflow's conditional branching depends on.
- **EDA extends, not replaces, existing job templates.** The same
  `Restore VM - *` job templates serve both the in-workflow restore
  (Chapter 6) and the event-driven restore (Chapter 8) — only the caller
  and the snapshot-lookup logic differ.
- **Build platform-specific first, consolidate second.** Writing the
  OpenShift Virtualization and VMware paths separately first (Chapters 4,
  6, 8) made the *actual* differences between them obvious — which made the
  Chapter 9 consolidation straightforward instead of guesswork.

## Where this goes next

- **Automation Analytics** — once this runs in production, it becomes the
  source for metrics like "automatic restores per month" and "mean time to
  recovery for patch failures" — quantifying the value of the self-healing
  loop.
- **More EDA use cases on the same pattern** — the same
  event → condition → `run_job_template` shape applies to other signals
  from this VM fleet: a disk-usage alert triggering a cleanup job, a
  certificate-expiry warning triggering a renewal playbook, or a failed
  backup triggering a retry with alternate storage.
- **Ansible Lightspeed** — as more platforms or roles are added (a third
  virtualization platform, a different OS family for patching), Lightspeed
  can help scaffold the new platform-specific roles from the same pattern
  established in Chapter 9.

The use case that opened this demo — *"back up a VM, patch it, and restore
it automatically if the patch fails, on two different virtualization
platforms"* — is now a single, auditable, self-healing workflow.

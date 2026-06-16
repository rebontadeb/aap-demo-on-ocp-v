# AAP Demo: Automated VM Backup, Patch & Restore

A chapter-wise, use-case-driven walkthrough of **Red Hat Ansible Automation
Platform (AAP)**. Every chapter builds on the artifacts created in the
previous one (same inventory, same credentials, same job templates), so the
demo reads as one continuous build-out rather than a set of disconnected
examples.

## The use case

> A VM administration team manages virtual machines on two platforms:
> **Red Hat OpenShift Virtualization** and **VMware vSphere**. Every patch
> cycle follows the same pattern:
>
> 1. **Back up** the VM (snapshot).
> 2. **Patch** the guest OS.
> 3. **Health-check** the VM after patching.
> 4. If the health check fails, **automatically restore** the VM from the
>    backup taken in step 1.
>
> This demo automates that entire loop with AAP — first as an on-demand
> workflow, then as a fully **event-driven, self-healing** process — and
> shows the same pattern implemented for both virtualization platforms.

## Table of contents

| # | Chapter | What it covers |
|---|---|---|
| 0 | [The Use Case](chapters/00-the-use-case.md) | The backup → patch → restore problem, and why it needs automation |
| 1 | [Introducing Ansible Automation Platform](chapters/01-introducing-aap.md) | What AAP is, high-level architecture, how it maps to this use case |
| 2 | [AAP Components, Explained](chapters/02-aap-components.md) | A standalone reference describing every AAP component |
| 3 | [Laying the Foundation](chapters/03-laying-the-foundation.md) | Organizations, credentials, inventories, projects, Execution Environments |
| 4 | [Automating the Backup](chapters/04-automated-backups.md) | Backup job templates for OpenShift Virtualization and VMware |
| 5 | [Automating the Patch](chapters/05-automated-patching.md) | The OS-patching playbook and post-patch health check |
| 6 | [The Safety Net: Workflow & Conditional Restore](chapters/06-the-safety-net.md) | Stitching backup → patch → health check → conditional restore |
| 7 | [Event-Driven Ansible, Explained](chapters/07-event-driven-ansible.md) | EDA architecture: rulebooks, rules, sources, conditions, actions |
| 8 | [EDA Use Case: Self-Healing VMs](chapters/08-eda-self-healing-usecase.md) | Auto-restore triggered by a monitoring alert, independent of the workflow run |
| 9 | [Two Platforms, One Process](chapters/09-two-platforms-one-process.md) | Consolidating OpenShift Virtualization and VMware into one automation pattern |
| 10 | [Conclusion & Outcomes](chapters/10-conclusion.md) | Results, lessons learned, what's next |

## How to use this demo

- This is **documentation, not a runnable repo**. Every chapter includes
  illustrative YAML (playbooks, rulebooks, workflow diagrams) that mirrors
  what you would actually build in AAP, but is meant for explanation, not
  execution as-is.
- Diagrams use [Mermaid](https://mermaid.js.org/) syntax and render directly
  on GitHub/GitLab or in any Mermaid-aware Markdown viewer.
- Terminology follows **Ansible Automation Platform 2.5** (Automation
  Controller, Private Automation Hub, EDA Controller, Execution
  Environments, Decision Environments, Automation Mesh).
- The same inventory group names (`ocpvirt_vms`, `vmware_vms`) and job
  template names introduced in Chapter 3 are reused throughout, so later
  chapters always refer back to something already established.

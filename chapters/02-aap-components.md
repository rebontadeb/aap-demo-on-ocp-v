# Chapter 2: AAP Components, Explained

This chapter is a standalone reference. It walks through every major
component of Red Hat Ansible Automation Platform, what it does, and —
briefly — how it will be used later in this demo. If you already know AAP's
architecture, skim the table at the end and move on to
[Chapter 3](03-laying-the-foundation.md).

---

## 1. Ansible Core (the engine)

Ansible Core is the open-source automation engine: the `ansible` and
`ansible-playbook` command-line tools, the module library, and the YAML
playbook language itself. Everything else in AAP is built around this
engine — AAP doesn't replace Ansible, it operationalizes it.

**In this demo:** every backup, patch, and restore task is a normal Ansible
task using normal modules (`community.vmware.*`, `kubernetes.core.k8s`,
`ansible.builtin.dnf`, etc.). AAP is the layer that runs, schedules,
secures, and audits those playbooks.

---

## 2. Automation Controller

Automation Controller (the successor to "Ansible Tower") is the control
plane of AAP — the web UI, REST API, and scheduler that everything else
plugs into. It provides:

- **Job Templates** — a saved, reusable definition of "run this playbook,
  from this project, against this inventory, with these credentials."
- **Workflow Job Templates** — chains of job templates (and other
  workflows) with branching logic based on success/failure.
- **Inventories** — static or dynamically-sourced lists of hosts/VMs,
  organized into groups.
- **Credentials** — securely stored secrets (API tokens, SSH keys, vCenter
  logins) injected into jobs at runtime, never exposed in playbooks.
- **RBAC** — organizations, teams, and users with fine-grained permissions
  on who can view, launch, or edit which templates.
- **Scheduling** — run job templates and workflows on a recurring schedule
  (e.g., a monthly patch window).
- **Surveys** — simple forms that prompt for input (e.g., "which VM group
  do you want to patch?") when a job is launched.
- **Audit/job history** — every run is logged: who launched it, what
  changed, full output, success/failure.

**In this demo:** Automation Controller hosts the **Backup**, **Patch**,
and **Restore** job templates for both platforms, plus the **workflow** that
ties them together (Chapter 6). It's the single place where the VM admin
team launches, schedules, and audits the whole patch cycle.

---

## 3. Execution Environments (EE)

An Execution Environment is a **container image** that bundles:

- Ansible Core (a specific version)
- Python (a specific version)
- The Ansible **collections** a playbook needs (e.g.,
  `kubernetes.core`, `community.vmware`)
- Any system or Python dependencies those collections require

Jobs in Automation Controller run *inside* an Execution Environment
container, rather than directly on the controller host. This means a job's
runtime dependencies are fully defined, version-pinned, and portable —
"it works on my machine" stops being a problem because the "machine" is the
container image itself.

EEs are built with the `ansible-builder` tool from a small definition file
(`execution-environment.yml`) and pushed to a registry — typically Private
Automation Hub.

**In this demo:** two Execution Environments are built —
`ee-ocpvirt` (with `kubernetes.core` and the OpenShift Virtualization
collections) and `ee-vmware` (with `community.vmware`). Each job template
in Chapters 4–6 specifies which EE it needs, so the right tooling is always
present, regardless of which platform a job is targeting.

---

## 4. Decision Environments (DE)

A Decision Environment is the Event-Driven Ansible equivalent of an
Execution Environment: a container image bundling `ansible-rulebook`, the
Python event-source plugins, and any collections a rulebook's actions need
(e.g., the collection containing `run_job_template`).

**In this demo:** the self-healing rulebook in Chapter 8 runs inside a DE
that includes the webhook source plugin and the `ansible.eda` action
collection.

---

## 5. Event-Driven Ansible (EDA)

Event-Driven Ansible adds a **reactive layer** on top of Automation
Controller. Instead of (or in addition to) running jobs on a schedule or
on demand, EDA listens for events from external systems — monitoring
alerts, webhooks, message queues, log streams — and responds automatically
according to rules defined in a **rulebook**.

The **EDA Controller** is the management layer: it manages **rulebook
activations** (running rulebooks), much like Automation Controller manages
job templates. EDA is covered in full depth in
[Chapter 7](07-event-driven-ansible.md) and [Chapter 8](08-eda-self-healing-usecase.md).

**In this demo:** EDA is what turns this from "an automated workflow you
have to run" into "a system that heals itself" — it watches for VMs that
become unhealthy *after* a patch (even hours later) and automatically
triggers the restore job template.

---

## 6. Automation Mesh

Automation Mesh is AAP's distributed networking layer, built on a
lightweight overlay tool called **Receptor**. It lets job execution happen
on nodes spread across data centers, clouds, and network segments, while
still being managed from one Automation Controller.

Node types include:

- **Control nodes** — run the controller services themselves.
- **Hybrid nodes** — control + execution.
- **Execution nodes** — run jobs (EE containers) close to the
  infrastructure they manage.
- **Hop nodes** — relay traffic between segmented networks without needing
  direct connectivity from the control plane to every execution node.

**In this demo:** the VMware vCenter environment and the OpenShift
Virtualization cluster may sit in different network zones (e.g., a
traditional data center vs. a newer Kubernetes platform). Automation Mesh
lets a single Automation Controller reach both — placing execution nodes
near each platform — without punching new firewall holes for every job.

---

## 7. Private Automation Hub

Private Automation Hub is an on-premises (or private cloud) repository for
**automation content**:

- Ansible **collections** — both Red Hat-certified and custom/internal
- **Execution Environment and Decision Environment images**
- Content **namespaces and access controls**, so teams publish and consume
  approved content only

It's the "single source of truth" for what automation content is approved
for use, version-pinned, and where EE/DE images are pulled from.

**In this demo:** Private Automation Hub stores the `ee-ocpvirt` and
`ee-vmware` Execution Environment images built in Chapter 3, plus any
custom collection containing shared roles (e.g., the patch-and-health-check
role used in Chapter 9).

---

## 8. Automation Content Navigator

Automation Content Navigator is a **terminal-based (TUI) development tool**
for building and testing Ansible content *inside* an Execution Environment
— the same environment a job will eventually run in. It lets a playbook
author step through plays and tasks, inspect inventories, browse module
documentation, and explore collections, all without needing a full
Automation Controller instance.

**In this demo:** the backup, patch, and restore playbooks (Chapters 4–6)
would be developed and tested with Automation Content Navigator against
`ee-ocpvirt` / `ee-vmware` *before* being added to the Project and wired up
as job templates — catching collection or dependency issues early.

---

## 9. Content Collections

Collections are the distribution format for Ansible content: modules,
roles, plugins, and documentation, versioned and shareable via Galaxy or
Automation Hub. Two collections are central to this demo:

- **`kubernetes.core`** (plus OpenShift Virtualization-specific content) —
  manages `VirtualMachine`, `VirtualMachineSnapshot`, and
  `VirtualMachineRestore` custom resources.
- **`community.vmware`** (or the newer `vmware.vmware` collection) —
  manages vSphere VM snapshots, power state, and guest operations.

**In this demo:** these collections are the actual "drivers" behind the
backup and restore job templates in Chapters 4 and 6 — they're what gets
baked into the two Execution Environments.

---

## 10. Automation Analytics

Automation Analytics is a SaaS-based reporting service that aggregates data
from Automation Controller about job runs, automation coverage, and
adoption trends — helping teams understand *where* automation is delivering
value (and where manual processes still dominate).

**In this demo:** once the backup/patch/restore workflow and EDA rulebook
are in production, Automation Analytics would show metrics like "number of
auto-remediated patch failures per month" and "mean time to restore" — the
kind of data referenced in [Chapter 10](10-conclusion.md).

---

## 11. Ansible Lightspeed

Ansible Lightspeed is an AI-assisted authoring tool (built on watsonx Code
Assistant) that helps write Ansible content — suggesting tasks, playbooks,
and roles from natural-language prompts inside supported IDEs.

**In this demo:** Lightspeed isn't part of the core workflow, but it's the
kind of tool that would speed up writing the platform-specific roles
introduced in Chapter 9 (e.g., scaffolding the VMware snapshot-revert
tasks from a prompt like "revert a vSphere VM to a named snapshot").

---

## Summary: component-to-use-case map

| Component | Role in the backup/patch/restore use case |
|---|---|
| Ansible Core | Runs every playbook's tasks |
| Automation Controller | Hosts backup/patch/restore job templates and the workflow |
| Execution Environments | `ee-ocpvirt` and `ee-vmware` — right tools for each platform |
| Decision Environment | Runs the EDA rulebook for self-healing (Ch. 8) |
| Event-Driven Ansible | Watches for post-patch failures and triggers restore automatically |
| Automation Mesh | Connects the control plane to both the OCP Virt cluster and vCenter |
| Private Automation Hub | Stores EE/DE images and shared collections/roles |
| Automation Content Navigator | Where the playbooks/roles are developed and tested |
| Content Collections | `kubernetes.core` and `community.vmware` — the actual API drivers |
| Automation Analytics | Reports on automation coverage and self-healing outcomes |
| Ansible Lightspeed | Accelerates authoring of platform-specific roles |

With the platform map established, the next chapter starts building: the
organization, credentials, inventory, projects, and Execution Environments
that everything else depends on.

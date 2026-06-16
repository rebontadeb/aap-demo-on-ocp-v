# Chapter 9: Two Platforms, One Process

Chapters 4, 6, and 8 built **the same logical pipeline twice** — once for
OpenShift Virtualization, once for VMware — because the underlying APIs are
genuinely different. This chapter steps back, lays both implementations
side by side, and then shows how to fold them into a single set of job
templates without losing the platform-specific detail.

## Side by side

| | OpenShift Virtualization | VMware vSphere |
|---|---|---|
| VM object | `VirtualMachine` custom resource (kubevirt.io) | VM object managed by vCenter |
| Backup primitive | `VirtualMachineSnapshot` custom resource | vSphere snapshot |
| Restore primitive | `VirtualMachineRestore` custom resource | Snapshot revert |
| Key collection | `kubernetes.core` | `community.vmware` |
| Inventory plugin | `kubernetes.core.k8s` inventory | `community.vmware.vmware_vm_inventory` |
| Credential type | Red Hat OpenShift or Kubernetes API Bearer Token | VMware vCenter |
| Inventory group | `ocpvirt_vms` | `vmware_vms` |
| `vm_platform` group var | `ocpvirt` | `vmware` |
| Execution Environment | `ee-ocpvirt` | `ee-vmware` |

Everything in this table is **inherently platform-specific** — that's not
duplication to "fix", it's the actual difference between the two APIs.
What *is* duplication is everything **around** these differences: two
backup job templates, two restore job templates, two workflows — all with
identical structure and identical failure-handling logic.

## Step 1: collapse backup and restore into platform-aware playbooks

The backup and restore *logic* (Chapters 4 and 6) becomes two small roles
per operation — `backup_ocpvirt` / `backup_vmware`,
`restore_ocpvirt` / `restore_vmware` — containing exactly the tasks already
written in those chapters. A thin top-level playbook picks the right role
based on the `vm_platform` group variable set back in
[Chapter 3](03-laying-the-foundation.md):

```yaml
# playbooks/backup.yml
---
- name: Backup VM (platform-aware)
  hosts: "{{ target_hosts | default('all') }}"
  gather_facts: false
  vars:
    snapshot_name: "pre-patch-{{ tower_job_id }}-{{ inventory_hostname }}"

  tasks:
    - name: Back up via OpenShift Virtualization
      ansible.builtin.include_role:
        name: backup_ocpvirt
      when: vm_platform == "ocpvirt"

    - name: Back up via VMware
      ansible.builtin.include_role:
        name: backup_vmware
      when: vm_platform == "vmware"

    - name: Record snapshot name for downstream steps
      ansible.builtin.set_stats:
        data:
          snapshot_name: "{{ snapshot_name }}"
        per_host: true
```

```yaml
# playbooks/restore.yml
---
- name: Restore VM (platform-aware)
  hosts: "{{ target_hosts | default('all') }}"
  gather_facts: false

  tasks:
    - name: Restore via OpenShift Virtualization
      ansible.builtin.include_role:
        name: restore_ocpvirt
      when: vm_platform == "ocpvirt"

    - name: Restore via VMware
      ansible.builtin.include_role:
        name: restore_vmware
      when: vm_platform == "vmware"
```

`playbooks/patch_and_healthcheck.yml` from Chapter 5 needs **no change** —
it was already platform-agnostic.

## Step 2: one Execution Environment

A single playbook that can run either branch needs an Execution Environment
containing **both** sets of collections:

```yaml
# execution-environment.yml (ee-vm-lifecycle)
version: 3
images:
  base_image:
    name: registry.redhat.io/ansible-automation-platform-25/ee-minimal-rhel9:latest
dependencies:
  galaxy: requirements.yml   # kubernetes.core, redhat.openshift_virtualization,
                              # community.vmware, vmware.vmware
  python: requirements.txt   # kubernetes, openshift, pyvmomi, requests
```

This is the one real tradeoff of consolidation: `ee-vm-lifecycle` is larger
than either `ee-ocpvirt` or `ee-vmware` alone. In exchange, there's one
image to build, scan, and patch instead of two.

## Step 3: one set of job templates, multiple credentials

A job template can hold **more than one credential**, as long as the types
don't conflict. `Backup VM` and `Restore VM` each carry both:

| Setting | `Backup VM` | `Restore VM` |
|---|---|---|
| Inventory | `VM Fleet` (any group) | `VM Fleet` (any group) |
| Playbook | `playbooks/backup.yml` | `playbooks/restore.yml` |
| Execution Environment | `ee-vm-lifecycle` | `ee-vm-lifecycle` |
| Credentials | `ocpvirt-api-token` **and** `vcenter-creds` | `ocpvirt-api-token` **and** `vcenter-creds` |

`Patch & Health-Check VM` from Chapter 5 is unchanged — it already worked
across both groups.

## Step 4: one workflow

The two near-identical workflows from Chapter 6 collapse into a single
**`VM Patch Cycle`**:

```mermaid
flowchart TD
    Start(["Launch: VM Patch Cycle\nlimit = target VM\n(vm_platform comes from\nthe VM's group membership)"]) --> Backup["Backup VM"]
    Backup -->|on success| Patch["Patch & Health-Check VM"]
    Backup -->|on failure| AlertBackup(["Notify: backup failed,\npatch skipped"])
    Patch -->|on success\n(healthy)| Done(["Done - VM patched\nand healthy"])
    Patch -->|on failure\n(unhealthy)| Restore["Restore VM"]
    Restore --> AlertRestore(["Notify: rolled back\nto pre-patch snapshot"])
```

Whether `limit` points at a VM in `ocpvirt_vms` or `vmware_vms`, the same
three job templates run — each one branching internally via `vm_platform`.
The EDA rulebook from Chapter 8 simplifies too: both `run_job_template`
rules now target the **same** `Restore VM` job template, differing only in
which VM (`limit`) they pass.

## What changed, what didn't

| | Before (Ch. 4–8) | After (Ch. 9) |
|---|---|---|
| Job templates | 6 (3 per platform) | 3 (platform-aware) |
| Workflows | 2 | 1 |
| Execution Environments | 2 (`ee-ocpvirt`, `ee-vmware`) | 1 (`ee-vm-lifecycle`) |
| EDA rules | 2 (one per platform) | 2 conditions → same job template |
| Platform-specific logic | Spread across separate templates | Isolated to roles, selected by `vm_platform` |

Nothing about *what* happens on each platform changed — the
`VirtualMachineSnapshot`/`VirtualMachineRestore` calls and the vSphere
snapshot calls are identical to Chapters 4 and 6. What changed is **where
the platform difference lives**: no longer in which job template you pick,
but in a single `when: vm_platform == ...` inside two small playbooks.
Adding a third platform later means adding a role and an inventory group —
not a third copy of every job template and workflow.

[Chapter 10](10-conclusion.md) wraps up with the overall results.

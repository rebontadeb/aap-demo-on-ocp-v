# Chapter 6: The Safety Net — Workflow & Conditional Restore

This chapter ties Chapters 4 and 5 together into a single **Workflow Job
Template** that implements the full use case from Chapter 0:
**backup → patch → health check → restore if unhealthy** — fully
unattended.

## Design note: one workflow run per VM

To keep the success/failure branching clean, the workflow is launched
**once per VM**, with `limit: <vm-name>` set at launch time (Chapter 8's
EDA rulebook does exactly this when it launches the workflow
automatically). This avoids ambiguity like "3 of 10 VMs in this batch
failed their health check — which ones get restored?" — each VM gets its
own pass/fail outcome and its own restore decision.

## The workflow

For OpenShift Virtualization, **`VM Patch Cycle - OpenShift Virtualization`**
chains the job templates from Chapters 4–5 plus a new restore job template:

```mermaid
flowchart TD
    Start([Launch: VM Patch Cycle\nlimit = target VM]) --> Backup["Backup VM -\nOpenShift Virtualization"]
    Backup -->|on success| Patch["Patch & Health-Check VM"]
    Backup -->|on failure| AlertBackup([Notify: backup failed,\npatch skipped])
    Patch -->|on success\n(healthy)| Done([Done - VM patched\nand healthy])
    Patch -->|on failure\n(unhealthy)| Restore["Restore VM -\nOpenShift Virtualization"]
    Restore --> AlertRestore([Notify: patch rolled back,\nVM restored from snapshot])
```

The VMware version, **`VM Patch Cycle - VMware`**, has the identical shape —
just swapping in `Backup VM - VMware`, `Restore VM - VMware`, and the
`vmware_vms` group. (Chapter 9 looks at collapsing these two near-identical
workflows into one.)

## Workflow nodes, in detail

| Step | Job Template | On success → | On failure → |
|---|---|---|---|
| 1 | `Backup VM - OpenShift Virtualization` | Step 2 | Notify "backup failed" — **stop** (no patch without a backup) |
| 2 | `Patch & Health-Check VM` | **Done** — VM patched and healthy | Step 3 |
| 3 | `Restore VM - OpenShift Virtualization` | Notify "rolled back to pre-patch snapshot" | Notify "restore *also* failed — page on-call" |

Note the failure path on step 1: if the backup itself fails, the workflow
stops *before* patching — there's no point patching a VM with no recovery
option.

## The new piece: restore job templates

### Restore — OpenShift Virtualization

| Setting | Value |
|---|---|
| Name | `Restore VM - OpenShift Virtualization` |
| Inventory | `VM Fleet`, limited to group `ocpvirt_vms` |
| Project | `vm-lifecycle-automation` |
| Playbook | `playbooks/restore_ocpvirt.yml` |
| Execution Environment | `ee-ocpvirt` |
| Credentials | `ocpvirt-api-token` |

```yaml
# playbooks/restore_ocpvirt.yml
---
- name: Restore OpenShift Virtualization VM from snapshot
  hosts: ocpvirt_vms
  gather_facts: false

  tasks:
    - name: Create VirtualMachineRestore for {{ inventory_hostname }}
      kubernetes.core.k8s:
        state: present
        definition:
          apiVersion: snapshot.kubevirt.io/v1beta1
          kind: VirtualMachineRestore
          metadata:
            name: "restore-{{ tower_job_id }}-{{ inventory_hostname }}"
            namespace: "{{ ocpvirt_namespace }}"
          spec:
            target:
              apiGroup: kubevirt.io
              kind: VirtualMachine
              name: "{{ inventory_hostname }}"
            virtualMachineSnapshotName: "{{ snapshot_name }}"

    - name: Wait for the restore to complete
      kubernetes.core.k8s_info:
        api_version: snapshot.kubevirt.io/v1beta1
        kind: VirtualMachineRestore
        name: "restore-{{ tower_job_id }}-{{ inventory_hostname }}"
        namespace: "{{ ocpvirt_namespace }}"
      register: restore_status
      until: restore_status.resources[0].status.complete | default(false)
      retries: 30
      delay: 10
```

### Restore — VMware

| Setting | Value |
|---|---|
| Name | `Restore VM - VMware` |
| Inventory | `VM Fleet`, limited to group `vmware_vms` |
| Project | `vm-lifecycle-automation` |
| Playbook | `playbooks/restore_vmware.yml` |
| Execution Environment | `ee-vmware` |
| Credentials | `vcenter-creds` |

```yaml
# playbooks/restore_vmware.yml
---
- name: Restore VMware VM from snapshot
  hosts: vmware_vms
  gather_facts: false

  tasks:
    - name: Revert {{ inventory_hostname }} to its pre-patch snapshot
      community.vmware.vmware_guest_snapshot:
        validate_certs: false
        datacenter: "{{ vcenter_datacenter }}"
        folder: "{{ vcenter_folder }}"
        name: "{{ inventory_hostname }}"
        state: revert
        snapshot_name: "{{ snapshot_name }}"
```

## How `snapshot_name` travels through the workflow

Both restore playbooks reference `{{ snapshot_name }}` — but never define
it. That's intentional: it's the **artifact** set by `set_stats` in the
backup playbook back in Chapter 4. Automation Controller automatically
passes artifacts from one workflow node to every node that runs after it,
as `extra_vars`. So:

1. **Backup** node creates `pre-patch-1042-vm-db-03` and sets
   `snapshot_name: pre-patch-1042-vm-db-03` as an artifact.
2. **Patch & Health-Check** node runs — doesn't need `snapshot_name`, but
   it's available if it did.
3. **Restore** node (only reached on failure) receives
   `snapshot_name: pre-patch-1042-vm-db-03` automatically, and restores
   exactly that snapshot — the one taken *for this run*, not some other
   stale snapshot.

## Result

At the end of this chapter, the use case from Chapter 0 is **fully
automated as an on-demand (or scheduled) workflow**: launch
`VM Patch Cycle - OpenShift Virtualization` (or the VMware equivalent) for
a VM, and it backs up, patches, health-checks, and — only if needed —
restores, with zero manual steps.

What it *can't* do yet: react to problems that show up **after** the
workflow has already finished — e.g., a service that crashes two hours
post-patch due to a delayed dependency issue. That's where
[Event-Driven Ansible](07-event-driven-ansible.md) comes in.

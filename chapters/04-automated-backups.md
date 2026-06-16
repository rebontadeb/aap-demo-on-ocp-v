# Chapter 4: Automating the Backup

The first step in the patch cycle is a backup. This chapter creates one job
template per platform, both producing the same outcome — **a snapshot that
can be restored from later** — using each platform's native primitive.

## Naming convention

Every snapshot is named using the **AAP job ID** that created it:

```
pre-patch-<tower_job_id>-<vm-name>
```

This makes every snapshot traceable back to a specific automation run in
Automation Controller's job history — useful for audits, and essential for
the conditional restore in Chapter 6, which needs to know *exactly* which
snapshot belongs to *this* patch run.

## Backup job template — OpenShift Virtualization

| Setting | Value |
|---|---|
| Name | `Backup VM - OpenShift Virtualization` |
| Inventory | `VM Fleet`, limited to group `ocpvirt_vms` |
| Project | `vm-lifecycle-automation` |
| Playbook | `playbooks/backup_ocpvirt.yml` |
| Execution Environment | `ee-ocpvirt` |
| Credentials | `ocpvirt-api-token` |

On OpenShift Virtualization, a backup is a **`VirtualMachineSnapshot`**
custom resource. The playbook creates one and waits for it to report
`readyToUse: true`:

```yaml
# playbooks/backup_ocpvirt.yml
---
- name: Backup OpenShift Virtualization VM (pre-patch snapshot)
  hosts: ocpvirt_vms
  gather_facts: false
  vars:
    snapshot_name: "pre-patch-{{ tower_job_id }}-{{ inventory_hostname }}"

  tasks:
    - name: Create VirtualMachineSnapshot for {{ inventory_hostname }}
      kubernetes.core.k8s:
        state: present
        definition:
          apiVersion: snapshot.kubevirt.io/v1beta1
          kind: VirtualMachineSnapshot
          metadata:
            name: "{{ snapshot_name }}"
            namespace: "{{ ocpvirt_namespace }}"
          spec:
            source:
              apiGroup: kubevirt.io
              kind: VirtualMachine
              name: "{{ inventory_hostname }}"

    - name: Wait for the snapshot to be ready
      kubernetes.core.k8s_info:
        api_version: snapshot.kubevirt.io/v1beta1
        kind: VirtualMachineSnapshot
        name: "{{ snapshot_name }}"
        namespace: "{{ ocpvirt_namespace }}"
      register: snap_status
      until: snap_status.resources[0].status.readyToUse | default(false)
      retries: 30
      delay: 10

    - name: Record snapshot name for the restore step
      ansible.builtin.set_stats:
        data:
          snapshot_name: "{{ snapshot_name }}"
        per_host: true
```

The `ocpvirt-api-token` credential authenticates `kubernetes.core.k8s`
against the cluster — no token appears anywhere in the playbook itself.

## Backup job template — VMware vSphere

| Setting | Value |
|---|---|
| Name | `Backup VM - VMware` |
| Inventory | `VM Fleet`, limited to group `vmware_vms` |
| Project | `vm-lifecycle-automation` |
| Playbook | `playbooks/backup_vmware.yml` |
| Execution Environment | `ee-vmware` |
| Credentials | `vcenter-creds` |

On VMware, a backup is a **vSphere snapshot**, created with
`community.vmware.vmware_guest_snapshot`:

```yaml
# playbooks/backup_vmware.yml
---
- name: Backup VMware VM (pre-patch snapshot)
  hosts: vmware_vms
  gather_facts: false
  vars:
    snapshot_name: "pre-patch-{{ tower_job_id }}-{{ inventory_hostname }}"

  tasks:
    - name: Create vSphere snapshot for {{ inventory_hostname }}
      community.vmware.vmware_guest_snapshot:
        validate_certs: false
        datacenter: "{{ vcenter_datacenter }}"
        folder: "{{ vcenter_folder }}"
        name: "{{ inventory_hostname }}"
        state: present
        snapshot_name: "{{ snapshot_name }}"
        description: "Pre-patch snapshot, AAP job {{ tower_job_id }}"
        quiesce: true
        memory_dump: false

    - name: Record snapshot name for the restore step
      ansible.builtin.set_stats:
        data:
          snapshot_name: "{{ snapshot_name }}"
        per_host: true
```

The `vcenter-creds` credential injects the vCenter hostname, username, and
password as connection variables that `community.vmware.*` modules read
automatically.

## Why `set_stats` matters

Both playbooks end with `set_stats`, registering `snapshot_name` as a
**per-host artifact**. In Chapter 6, this value flows from the backup job
into the workflow's later steps — specifically, the restore job needs to
know *which* snapshot to roll back to, and `set_stats` is how that
information is passed forward without hardcoding anything.

## Result

After this chapter, two independent, on-demand job templates exist:
`Backup VM - OpenShift Virtualization` and `Backup VM - VMware`. Either can
be run by itself (e.g., for an ad-hoc backup), but their real purpose is as
**step one of the workflow** built in Chapter 6.

Next: [Chapter 5](05-automated-patching.md) automates the patch itself —
and the health check that decides whether a restore is needed.

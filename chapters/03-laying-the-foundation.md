# Chapter 3: Laying the Foundation

Before any backup, patch, or restore job can run, AAP needs four things in
place: **who can do what** (organization/teams), **how to authenticate**
(credentials), **what to act on** (inventory), and **with what tooling**
(projects + Execution Environments). This chapter sets all four up — every
later chapter refers back to the names defined here.

## 1. Organization and teams

A single organization, **`VM Operations`**, contains everything in this
demo. Inside it, two teams reflect the two platform specialties:

| Team | Responsibilities |
|---|---|
| `ocpvirt-admins` | Backup/patch/restore job templates targeting OpenShift Virtualization |
| `vmware-admins` | Backup/patch/restore job templates targeting VMware vSphere |

Both teams can see and launch the **workflow job template** built in
Chapter 6 — but each only has edit access to the job templates and
credentials for their own platform. This is standard AAP RBAC: access is
scoped per-resource, not all-or-nothing.

## 2. Credentials

Three credential types are created, each scoped to the team(s) that need
them:

| Credential name | Type | Used by |
|---|---|---|
| `ocpvirt-api-token` | *Red Hat OpenShift or Kubernetes API Bearer Token* | Backup/restore job templates for OpenShift Virtualization — authenticates to the cluster API to manage `VirtualMachineSnapshot` / `VirtualMachineRestore` resources |
| `vcenter-creds` | *VMware vCenter* | Backup/restore job templates for VMware — authenticates to vCenter to manage VM snapshots |
| `vm-guest-ssh` | *Machine (SSH private key)* | Patch job template — logs into the guest OS of any VM, on either platform, to apply OS patches |

Credentials are stored encrypted in AAP and **injected at runtime** — the
playbooks in later chapters never see a raw password or token; they
reference the credential by name in the job template definition.

## 3. Inventory

A single inventory, **`VM Fleet`**, holds both platforms as separate
groups:

```
VM Fleet
├── ocpvirt_vms      (sourced from the OpenShift Virtualization cluster)
└── vmware_vms       (sourced from vCenter)
```

Each group is populated by a **dynamic inventory source** rather than a
static host list, so newly created VMs are picked up automatically:

- `ocpvirt_vms` — uses the `kubernetes.core.k8s` inventory plugin, scoped to
  `VirtualMachine` / `VirtualMachineInstance` resources in the relevant
  namespace(s).
- `vmware_vms` — uses the `community.vmware.vmware_vm_inventory` plugin,
  pointed at the vCenter instance and a folder/cluster filter.

Each group also carries a **group variable** that the patch and restore
playbooks use later to decide which platform-specific logic to run
(this becomes important in [Chapter 9](09-two-platforms-one-process.md)):

```yaml
# group_vars/ocpvirt_vms.yml
vm_platform: ocpvirt
ocpvirt_namespace: vm-workloads

# group_vars/vmware_vms.yml
vm_platform: vmware
vcenter_datacenter: DC1
vcenter_folder: /DC1/vm/Production
```

## 4. Projects

A **Project** in AAP points Automation Controller at a Git repository
containing all the playbooks, roles, and (later) rulebooks used in this
demo:

```
vm-lifecycle-automation/
├── playbooks/
│   ├── backup_ocpvirt.yml
│   ├── backup_vmware.yml
│   ├── patch_and_healthcheck.yml
│   ├── restore_ocpvirt.yml
│   └── restore_vmware.yml
├── roles/
│   └── ...
└── rulebooks/
    └── self_heal_vm.yml
```

Automation Controller syncs this project on a schedule (or via webhook from
the Git server), so every job template always runs against the latest
committed version of these playbooks — and every change to the automation
itself is code-reviewed and version-controlled.

## 5. Execution Environments

Two Execution Environments are built with `ansible-builder`, one per
platform, each containing only the collections that platform's playbooks
need.

**`ee-ocpvirt`** — for OpenShift Virtualization job templates:

```yaml
# execution-environment.yml (ee-ocpvirt)
version: 3
images:
  base_image:
    name: registry.redhat.io/ansible-automation-platform-25/ee-minimal-rhel9:latest
dependencies:
  galaxy: requirements.yml   # kubernetes.core, redhat.openshift_virtualization
  python: requirements.txt   # kubernetes, openshift
```

**`ee-vmware`** — for VMware job templates:

```yaml
# execution-environment.yml (ee-vmware)
version: 3
images:
  base_image:
    name: registry.redhat.io/ansible-automation-platform-25/ee-minimal-rhel9:latest
dependencies:
  galaxy: requirements.yml   # community.vmware, vmware.vmware
  python: requirements.txt   # pyvmomi, requests
```

Both images are built, tagged, and pushed to **Private Automation Hub**
(Chapter 2), where Automation Controller pulls them when running job
templates.

## What's in place now

| Piece | Name(s) |
|---|---|
| Organization | `VM Operations` |
| Teams | `ocpvirt-admins`, `vmware-admins` |
| Credentials | `ocpvirt-api-token`, `vcenter-creds`, `vm-guest-ssh` |
| Inventory | `VM Fleet` → groups `ocpvirt_vms`, `vmware_vms` |
| Project | `vm-lifecycle-automation` (Git) |
| Execution Environments | `ee-ocpvirt`, `ee-vmware` |

With the foundation in place, Chapter 4 builds the first job templates:
**backup**.

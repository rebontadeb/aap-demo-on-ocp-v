# Inventory

A single static inventory, `hosts.yml`, with the same two groups used
throughout the narrative chapters: `ocpvirt_vms` (executed) and
`vmware_vms` (reference only).

## How `ansible_host` / `ansible_port` work here

Because Cluster A (AAP) and Cluster B (OpenShift Virtualization) are
separate infrastructure, each VM in `ocpvirt_vms` is reached via a NodePort
`Service` created in [`00-prerequisites.md`](../00-prerequisites.md):

```yaml
rhel9-app-01:
  ansible_host: 203.0.113.10   # any Cluster B node's IP
  ansible_port: 30022          # NodePort that forwards to rhel9-app-01:22
```

Only `03-playbooks/patch_and_healthcheck.yml` actually connects over SSH
using these values — `backup_ocpvirt.yml` and `restore_ocpvirt.yml` run
with `connection: local` and talk to the Kubernetes API instead (see
[`03-playbooks/README.md`](../03-playbooks/README.md)).

## Why there's no explicit API connection info

You won't find a `vcenter_host` or a Kubernetes API URL/token anywhere in
this inventory. That's intentional — both come from **credentials**
attached to the job templates in
[`05-controller-as-code/`](../05-controller-as-code/):

- The **`ocpvirt-api-token`** credential (type *Red Hat OpenShift or
  Kubernetes API Bearer Token*) injects `K8S_AUTH_HOST`,
  `K8S_AUTH_API_KEY`, and `K8S_AUTH_VERIFY_SSL` as environment variables.
  `kubernetes.core.k8s*` modules read these automatically — no `host` /
  `api_key` arguments needed in the playbooks.
- The **`vcenter-creds`** credential (type *VMware vCenter*, reference
  only) would inject `VMWARE_HOST`, `VMWARE_USER`, `VMWARE_PASSWORD`, and
  `VMWARE_VALIDATE_CERTS` for `community.vmware.*` modules the same way.

This keeps connection secrets out of both the inventory and the playbooks —
they live only in AAP's encrypted credential store.

## `vm_platform`

`group_vars/ocpvirt_vms.yml` and `group_vars/vmware_vms.yml` each set
`vm_platform`, exactly as introduced in
[Chapter 9](../../chapters/09-two-platforms-one-process.md) of the
narrative — it's not used by the executed playbooks directly (each
platform has its own playbook here rather than a consolidated
role-dispatch playbook), but it documents which platform each group
belongs to and is available to any future consolidation.

## Loading this inventory into AAP

`05-controller-as-code/configure_controller.yml` creates an
**SCM inventory source** pointing at `implementation/02-inventory/hosts.yml`
in this Git repository, so Automation Controller's `VM Fleet` inventory
stays in sync with this file automatically on every project update.

# Playbooks

| File | Status | Connects to |
|---|---|---|
| `backup_ocpvirt.yml` | **Executed** | Cluster B's K8s API (via `connection: local` + `ocpvirt-api-token` credential) |
| `patch_and_healthcheck.yml` | **Executed** | The VM's guest OS over SSH (via NodePort + `vm-guest-ssh` credential) |
| `restore_ocpvirt.yml` | **Executed** | Cluster B's K8s API |
| `backup_vmware.yml` | Reference only | vCenter API (not applicable - no VMware infra) |
| `restore_vmware.yml` | Reference only | vCenter API (not applicable - no VMware infra) |

## Two connection models, on purpose

- **`backup_ocpvirt.yml` and `restore_ocpvirt.yml`** set `connection: local`.
  They never SSH to the VM — they create/inspect `VirtualMachineSnapshot`
  and `VirtualMachineRestore` **custom resources** on Cluster B's
  Kubernetes API, using `kubernetes.core.k8s*`. `connection: local` means
  these tasks run in the AAP Execution Environment pod itself (on
  Cluster A), which is where the `ocpvirt-api-token` credential's
  `K8S_AUTH_*` environment variables are injected.

- **`patch_and_healthcheck.yml`** has no `connection` override — it uses
  the default SSH connection to `inventory_hostname:ansible_port`
  (the NodePort from [`02-inventory/`](../02-inventory/)), because patching
  genuinely happens *inside* the VM's guest OS.

## How `snapshot_name` and `patch_healthy` flow between them

Both are set via `ansible.builtin.set_stats` with `per_host: true`. When
these playbooks run as **workflow nodes** (configured in
[`05-controller-as-code/`](../05-controller-as-code/)), Automation
Controller automatically passes these as `extra_vars` to subsequent nodes:

1. `backup_ocpvirt.yml` sets `snapshot_name: pre-patch-<job_id>-<vm>`.
2. `patch_and_healthcheck.yml` sets `patch_healthy: true|false` and, if
   `false`, also fails the job — which is what makes the workflow take the
   "failure" branch to the restore node.
3. `restore_ocpvirt.yml` receives `snapshot_name` from step 1 (when run as
   part of the workflow) **or** looks up the latest `pre-patch-*` snapshot
   itself (when triggered independently by the EDA rulebook in
   [`04-rulebooks/`](../04-rulebooks/)).

## Running these by hand (outside AAP), for testing

With `ansible.controller`'s `K8S_AUTH_*` env vars exported manually and the
inventory's SSH details reachable:

```bash
export K8S_AUTH_HOST="https://api.cluster-b.example.com:6443"
export K8S_AUTH_API_KEY="<token from 00-prerequisites.md>"
export K8S_AUTH_VERIFY_SSL=false

ansible-playbook -i ../02-inventory/hosts.yml backup_ocpvirt.yml --limit rhel9-app-01
ansible-playbook -i ../02-inventory/hosts.yml patch_and_healthcheck.yml --limit rhel9-app-01
ansible-playbook -i ../02-inventory/hosts.yml restore_ocpvirt.yml --limit rhel9-app-01
```

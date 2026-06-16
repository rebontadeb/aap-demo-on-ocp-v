# Chapter 0 (Implementation): Prerequisites

Everything in this directory assumes **two independent OpenShift clusters**.
Nothing here assumes shared networking, a shared kubeconfig, or a shared
filesystem between them — every cross-cluster interaction is either an
HTTPS call to an API or an SSH connection to a NodePort.

## Cluster A — runs Ansible Automation Platform

1. **AAP 2.5 installed via the AAP Operator**, with Automation Controller,
   EDA Controller, and Private Automation Hub routes reachable.
2. **Outbound network access from Cluster A to Cluster B** on:
   - `443`/`6443` — Cluster B's Kubernetes/OpenShift API (for backup/restore)
   - The NodePort range (default `30000-32767`) — for SSH to VMs (for
     patching)
3. A machine (laptop, bastion, or the AAP control node itself) with:
   - `ansible-core` and `ansible-builder` (to build the Execution
     Environments in `01-execution-environments/`)
   - The `ansible.controller` and `ansible.eda` collections (to run the
     setup playbooks in `05-controller-as-code/`)
4. **Two AAP API tokens** (Personal Access Tokens), used only by the
   one-time setup playbooks:
   - One for **Automation Controller** (`configure_controller.yml`)
   - One for **EDA Controller** (`configure_eda.yml`)
5. **This Git repository** (or a fork of it) reachable from Cluster A, used
   as the AAP **Project** source — Automation Controller and EDA Controller
   both sync playbooks/rulebooks from it.

## Cluster B — runs OpenShift Virtualization

1. **OpenShift Virtualization (CNV) installed**, with one or more VMs
   running in a namespace — this demo uses `vm-workloads` and two example
   VMs, `rhel9-app-01` and `rhel9-db-01`.

2. **A ServiceAccount AAP can authenticate as**, scoped to exactly the
   resources the backup/restore playbooks touch:

   ```yaml
   # cluster-b/aap-vm-operator-rbac.yaml
   apiVersion: v1
   kind: ServiceAccount
   metadata:
     name: aap-vm-operator
     namespace: vm-workloads
   ---
   apiVersion: rbac.authorization.k8s.io/v1
   kind: Role
   metadata:
     name: aap-vm-operator
     namespace: vm-workloads
   rules:
     - apiGroups: ["kubevirt.io"]
       resources: ["virtualmachines", "virtualmachineinstances"]
       verbs: ["get", "list", "watch"]
     - apiGroups: ["snapshot.kubevirt.io"]
       resources: ["virtualmachinesnapshots", "virtualmachinerestores"]
       verbs: ["get", "list", "watch", "create", "delete"]
   ---
   apiVersion: rbac.authorization.k8s.io/v1
   kind: RoleBinding
   metadata:
     name: aap-vm-operator
     namespace: vm-workloads
   subjects:
     - kind: ServiceAccount
       name: aap-vm-operator
       namespace: vm-workloads
   roleRef:
     kind: Role
     name: aap-vm-operator
     apiGroup: rbac.authorization.k8s.io
   ```

   Apply it, then mint a token:

   ```bash
   oc apply -f cluster-b/aap-vm-operator-rbac.yaml
   oc create token aap-vm-operator -n vm-workloads --duration=8760h
   ```

   This token, plus Cluster B's API URL (`oc whoami --show-server`), become
   the **`ocpvirt-api-token`** credential in
   [`05-controller-as-code/`](05-controller-as-code/).

3. **An SSH-reachable path to each VM**, for the patch job. KubeVirt VMs
   don't get an externally-routable IP by default, so each VM needs a
   `Service` exposing port 22:

   ```yaml
   # cluster-b/rhel9-app-01-ssh-service.yaml
   apiVersion: v1
   kind: Service
   metadata:
     name: rhel9-app-01-ssh
     namespace: vm-workloads
   spec:
     type: NodePort
     selector:
       kubevirt.io/domain: rhel9-app-01
     ports:
       - port: 22
         targetPort: 22
         nodePort: 30022
   ```

   Repeat per VM with a unique `nodePort`. The resulting
   `<cluster-B-node-ip>:<nodePort>` pairs are what
   [`02-inventory/hosts.yml`](02-inventory/hosts.yml) uses as
   `ansible_host` / `ansible_port`.

4. **An SSH key pair for guest-OS access.** The private key becomes the
   **`vm-guest-ssh`** credential in Automation Controller; the public key
   must already be in each VM's `~/.ssh/authorized_keys` (e.g., injected via
   `cloud-init` when the VM was created).

## VMware vSphere — not available in this environment

The VMware-equivalent files (`backup_vmware.yml`, `restore_vmware.yml`, the
`ee-vmware` Execution Environment, and the VMware rule in the EDA rulebook)
are included for **parity and reference only**. They follow the exact same
pattern as the OpenShift Virtualization files — same variable names
(`snapshot_name`, `vm_platform`), same job template naming convention — but
are **not applied or run** as part of this demo, since no vCenter
environment is available to target. Every such file is marked with a
comment at the top:

```yaml
# NOT EXECUTED - reference only. No VMware infrastructure available
# in this environment. Provided for parity with the OpenShift
# Virtualization implementation (see ../chapters/09-two-platforms-one-process.md).
```

With both clusters prepared, continue to
[`01-execution-environments/`](01-execution-environments/).

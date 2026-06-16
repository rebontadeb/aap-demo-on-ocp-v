# Running the Demo, End to End

This walks through the whole use case on real infrastructure: two
OpenShift clusters, AAP on Cluster A managing VMs on Cluster B
([`README.md`](README.md)). Each step links back to the directory that
implements it.

## Order of operations

| Step | What | Where |
|---|---|---|
| 1 | Confirm Cluster A / Cluster B prerequisites | [`00-prerequisites.md`](00-prerequisites.md) |
| 2 | Build and push `ee-ocpvirt`, `de-self-heal` | [`01-execution-environments/`](01-execution-environments/) |
| 3 | Push this repository to a Git remote AAP can reach | — |
| 4 | Configure AAP objects (org, credentials, project, inventory, job templates, workflow, EDA) | [`05-controller-as-code/`](05-controller-as-code/) |
| 5 | Run the **happy path**: backup → patch → healthy | below |
| 6 | Run the **failure path**: backup → patch (forced failure) → restore | below |
| 7 | Trigger the **EDA self-heal** rulebook independently | below |
| 8 | VMware path — not executed | below |

---

## Step 1: Prerequisites

Work through [`00-prerequisites.md`](00-prerequisites.md) on both clusters:
AAP installed on Cluster A, the `aap-vm-operator` RBAC + token and per-VM
SSH `Service` objects applied on Cluster B, and an SSH key pair for the
guest OS. Keep the token and private key handy — they go into
`controller_secrets.yml` in Step 4.

## Step 2: Build and push the Execution/Decision Environments

Follow [`01-execution-environments/README.md`](01-execution-environments/README.md)
to build and push `ee-ocpvirt` and `de-self-heal` to a registry Cluster A's
AAP can pull from. Note the two image references — they go into
`controller_objects.yml` as `ee_ocpvirt_image` and `de_self_heal_image`.

## Step 3: Push this repository

AAP's **Project** syncs playbooks, the inventory, and the rulebook from
Git. Push this repository (or a fork) to a remote reachable from Cluster A,
and note its URL for `project_scm_url`.

## Step 4: Configure AAP via controller-as-code

Follow [`05-controller-as-code/README.md`](05-controller-as-code/README.md):

```bash
cd 05-controller-as-code
ansible-galaxy collection install -r requirements.yml

# fill in vars/controller_auth.yml, vars/eda_auth.yml,
# vars/controller_secrets.yml (vault-encrypt it), and the placeholders
# in vars/controller_objects.yml

ansible-playbook configure_controller.yml --vault-password-file <path>
ansible-playbook configure_eda.yml
```

After this, Automation Controller has:

- Organization `VM Operations`, credentials `ocpvirt-api-token` and
  `vm-guest-ssh`, project `vm-lifecycle-automation`, inventory `VM Fleet`
  (synced from `02-inventory/hosts.yml`).
- Job templates `Backup VM - OpenShift Virtualization`,
  `Patch & Health-Check VM`, `Restore VM - OpenShift Virtualization`.
- Workflow `VM Patch Cycle - OpenShift Virtualization`:
  `backup` → (success) → `patch` → (failure) → `restore`
  ([Chapter 6](../chapters/06-the-safety-net.md)).

And EDA Controller has the `de-self-heal` decision environment and an
active **Rulebook Activation** running
[`04-rulebooks/self_heal_vm.yml`](04-rulebooks/self_heal_vm.yml)
([Chapter 8](../chapters/08-eda-self-healing-usecase.md)).

---

## Step 5: Happy path — backup, patch, healthy

Launch `VM Patch Cycle - OpenShift Virtualization`, limited to one VM. From
the AAP UI: **Templates → VM Patch Cycle - OpenShift Virtualization →
Launch**, set **Limit** to `rhel9-app-01`. Or via the API:

```bash
curl -s -X POST \
  -H "Authorization: Bearer $CONTROLLER_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"limit": "rhel9-app-01"}' \
  "$CONTROLLER_HOST/api/v2/workflow_job_templates/<id>/launch/"
```

(Find `<id>` via `GET /api/v2/workflow_job_templates/?name=VM+Patch+Cycle+-+OpenShift+Virtualization`.)

Expected outcome:

1. **`backup` node** runs `backup_ocpvirt.yml` — creates a
   `VirtualMachineSnapshot` named `pre-patch-<job_id>-rhel9-app-01` and sets
   the `snapshot_name` artifact. Verify on Cluster B:

   ```bash
   oc get virtualmachinesnapshot -n vm-workloads -l vm-name=rhel9-app-01
   ```

2. **`patch` node** runs `patch_and_healthcheck.yml` over SSH — updates
   packages, reboots if needed, and checks `httpd` (`health_check_service`)
   and port `443` (`health_check_port`). Both pass, so `patch_healthy: true`
   and the node **succeeds**.

3. The `restore` node is **skipped** (only reachable on `patch` failure).
   The workflow finishes green.

## Step 6: Failure path — forced patch failure, automatic restore

To exercise [Chapter 6](../chapters/06-the-safety-net.md)'s safety net
deterministically, launch the same workflow but override
`health_check_service` to a service that doesn't exist, so the post-patch
health check fails on purpose:

```bash
curl -s -X POST \
  -H "Authorization: Bearer $CONTROLLER_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"limit": "rhel9-app-01", "extra_vars": {"health_check_service": "definitely-not-a-real-service"}}' \
  "$CONTROLLER_HOST/api/v2/workflow_job_templates/<id>/launch/"
```

Expected outcome:

1. **`backup` node** succeeds as before — new `pre-patch-<job_id>-rhel9-app-01`
   snapshot, `snapshot_name` artifact set.
2. **`patch` node** patches and reboots normally, but the systemd check for
   `definitely-not-a-real-service` can't be `active` — `patch_healthy: false`,
   and the task `Fail this host if the health check did not pass` fails the
   node.
3. Because `patch` failed, the workflow follows the **`failure_nodes`**
   edge to **`restore`**, which runs `restore_ocpvirt.yml`. It receives
   `snapshot_name` as a workflow artifact from step 1 (no lookup needed),
   creates a `VirtualMachineRestore` CR, and waits for
   `status.complete`. Verify on Cluster B:

   ```bash
   oc get virtualmachinerestore -n vm-workloads
   ```

The workflow as a whole shows `patch` failed and `restore` succeeded —
exactly the "backup → patch → restore-on-failure" use case from
[Chapter 0](../chapters/00-the-use-case.md), fully automated.

---

## Step 7: EDA self-heal — a failure the workflow never sees

[Chapter 8](../chapters/08-eda-self-healing-usecase.md)'s scenario: a VM
passed its post-patch health check, but a service crashes **hours later** —
no workflow is running to notice. The active rulebook activation from Step
4 is listening for exactly this.

1. In the AAP UI, open **Automation Decisions → Rulebook Activations →
   Self-heal VMs after delayed patch failures** and find its webhook URL
   (and any auth token it requires). The exact location of this URL depends
   on your installed AAP/EDA version — check the activation's details page
   or `GET /api/eda/v1/activations/`.

2. POST an Alertmanager-style payload that matches the rule's condition in
   [`04-rulebooks/self_heal_vm.yml`](04-rulebooks/self_heal_vm.yml):

   ```bash
   curl -X POST "<activation webhook URL>" \
     -H "Content-Type: application/json" \
     -d '{
       "status": "firing",
       "labels": {
         "alertname": "VMServiceDown",
         "platform": "ocpvirt",
         "vm_name": "rhel9-app-01"
       }
     }'
   ```

3. Expected outcome: the rule matches and calls `run_job_template` for
   `Restore VM - OpenShift Virtualization` with `limit: rhel9-app-01`. This
   time `snapshot_name` is **not** supplied (there's no workflow run behind
   this trigger), so `restore_ocpvirt.yml`'s lookup block finds the most
   recent `pre-patch-*` `VirtualMachineSnapshot` for `rhel9-app-01` (from
   Step 5 or 6) and restores that. Verify:

   - Automation Controller → Jobs: a new `Restore VM - OpenShift
     Virtualization` job, launched by the EDA activation, with
     `limit=rhel9-app-01`.
   - Cluster B: a new `VirtualMachineRestore` CR, same as in Step 6.

This is the same restore job template used in Steps 5–6 — EDA just calls it
from an independent trigger, hours after any workflow would have finished.

---

## Step 8: VMware — not executed

No VMware/vCenter environment is available, so this path is **reference
only** ([Chapter 9](../chapters/09-two-platforms-one-process.md)). The
files exist and follow the identical pattern:

| OpenShift Virtualization (executed) | VMware (reference only) |
|---|---|
| `01-execution-environments/ee-ocpvirt/` | `01-execution-environments/ee-vmware/` |
| `02-inventory` group `ocpvirt_vms` | `02-inventory` group `vmware_vms` |
| `03-playbooks/backup_ocpvirt.yml`, `restore_ocpvirt.yml` | `03-playbooks/backup_vmware.yml`, `restore_vmware.yml` |
| Credential `ocpvirt-api-token` | Credential `vcenter-creds` |
| Job templates / workflow `... - OpenShift Virtualization` | Job templates / workflow `... - VMware` |
| `04-rulebooks/self_heal_vm.yml` rule 1 (`platform == "ocpvirt"`) | `04-rulebooks/self_heal_vm.yml` rule 2 (`platform == "vmware"`) |

If a vCenter environment becomes available: fill in the VMware fields in
`controller_secrets.yml` and the placeholders in `controller_objects.yml`,
set `apply_vmware_objects: true` in
[`05-controller-as-code/configure_controller.yml`](05-controller-as-code/configure_controller.yml),
re-run it, and re-run `configure_eda.yml`. Steps 5–7 above then apply
unchanged to `vmw-app-01` and the `VM Patch Cycle - VMware` workflow —
`Patch & Health-Check VM` is reused as-is, since patching is platform-agnostic.

---

## Cleaning up

- **Cluster B**: `oc delete virtualmachinesnapshot,virtualmachinerestore -n vm-workloads -l vm-name=rhel9-app-01`
  removes the snapshots/restores created while testing.
- **EDA Controller**: disable or delete the rulebook activation
  (`ansible.eda.rulebook_activation` with `state: absent`, or via the UI)
  to stop it listening.
- **Automation Controller**: re-running `configure_controller.yml` is
  idempotent and won't recreate anything already deleted manually — to fully
  remove the demo's objects, set `state: absent` on the relevant tasks in
  [`05-controller-as-code/configure_controller.yml`](05-controller-as-code/configure_controller.yml)
  and re-run.

# Controller-as-Code

Two idempotent playbooks that create **every AAP object** this demo uses,
using the `ansible.controller` and `ansible.eda` collections - no manual UI
clicking. Run from a machine with network access to Cluster A's AAP routes
(see [`../00-prerequisites.md`](../00-prerequisites.md), Cluster A, item 3).

| File | Creates |
|---|---|
| `configure_controller.yml` | Organization, credentials, project, execution environments, inventory + SCM inventory source, job templates, workflow job template + nodes |
| `configure_eda.yml` | EDA project, `de-self-heal` decision environment, rulebook activation |

## Files

```
05-controller-as-code/
├── requirements.yml                    # ansible.controller, ansible.eda
├── vars/
│   ├── controller_auth.yml.example     # -> controller_auth.yml
│   ├── eda_auth.yml.example            # -> eda_auth.yml
│   ├── controller_secrets.yml.example  # -> controller_secrets.yml (vault!)
│   └── controller_objects.yml          # non-secret object definitions
├── tasks/
│   └── workflow_with_nodes.yml         # included per-workflow
├── configure_controller.yml
└── configure_eda.yml
```

## 1. Install the collections

```bash
ansible-galaxy collection install -r requirements.yml
```

## 2. Fill in the variable files

Copy each `.example` file, dropping the `.example` suffix, and fill in real
values:

```bash
cd vars
cp controller_auth.yml.example   controller_auth.yml
cp eda_auth.yml.example          eda_auth.yml
cp controller_secrets.yml.example controller_secrets.yml
```

- **`controller_auth.yml`** / **`eda_auth.yml`** — AAP route + API token.
  Both default to reading the token from an environment variable
  (`CONTROLLER_TOKEN`, `EDA_CONTROLLER_TOKEN`) so the file itself holds no
  secret.
- **`controller_secrets.yml`** — Cluster B's API host/token and the VM
  guest SSH private key (from
  [`../00-prerequisites.md`](../00-prerequisites.md)). Encrypt it:

  ```bash
  ansible-vault encrypt controller_secrets.yml
  ```

  Every task in `configure_controller.yml` that consumes these values runs
  with `no_log: true`.
- **`controller_objects.yml`** — not secret, but **edit the placeholders**:
  `project_scm_url` (this repo or your fork), `ee_ocpvirt_image`, and
  `de_self_heal_image` (pushed in
  [`../01-execution-environments/`](../01-execution-environments/)).

None of `controller_auth.yml`, `eda_auth.yml`, or `controller_secrets.yml`
should be committed - add them to `.gitignore`.

## 3. Run

```bash
ansible-playbook configure_controller.yml --vault-password-file <path>
ansible-playbook configure_eda.yml
```

Both are **idempotent** - re-running after editing `controller_objects.yml`
updates existing objects rather than duplicating them.

## The `apply_vmware_objects` flag

`configure_controller.yml` defines `apply_vmware_objects: false`. Every
object in `controller_objects.yml` marked `_executed: false` (the
`vcenter-creds` credential, `ee-vmware`, the two VMware job templates, and
the `VM Patch Cycle - VMware` workflow) is skipped while this is `false` -
which it is throughout this demo, since no VMware infrastructure is
available. If a real vCenter environment is later connected, fill in the
VMware fields in `controller_secrets.yml` and `controller_objects.yml`, set
`apply_vmware_objects: true`, and re-run.

## Why one job template serves both platforms

`Patch & Health-Check VM` appears in **both** the OpenShift Virtualization
and VMware workflows in `controller_objects.yml`. Patching happens over SSH
to the guest OS ([`../03-playbooks/README.md`](../03-playbooks/README.md)),
which is identical regardless of hypervisor - only the backup/restore steps
differ per platform.

## What `configure_controller.yml` does, in order

1. **Organization** — `VM Operations` (`org_name`).
2. **Credentials** — `ocpvirt-api-token` ("OpenShift or Kubernetes API
   Bearer Token", injects `K8S_AUTH_*`) and `vm-guest-ssh` ("Machine").
3. **Project** — `vm-lifecycle-automation`, an SCM project pointing at this
   Git repository, used as the source for playbooks, the inventory, and
   (via `configure_eda.yml`) the rulebook.
4. **Execution environment** — registers `ee-ocpvirt`.
5. **Inventory** — `VM Fleet`, with an **SCM inventory source** synced from
   `implementation/02-inventory/hosts.yml` in the project.
6. **Job templates** — `Backup VM - OpenShift Virtualization`,
   `Patch & Health-Check VM`, `Restore VM - OpenShift Virtualization`, each
   with `ask_limit_on_launch: true` so a single VM can be targeted at
   launch.
7. **Workflow** — `VM Patch Cycle - OpenShift Virtualization`:
   `backup` → (success) → `patch` → (failure) → `restore`, matching
   [Chapter 6](../../chapters/06-the-safety-net.md).

## What `configure_eda.yml` does

Registers the rulebook from
[`../04-rulebooks/self_heal_vm.yml`](../04-rulebooks/self_heal_vm.yml) as a
continuously-running **Rulebook Activation** in EDA Controller — the
production equivalent of the manual `ansible-rulebook` test described in
[`../04-rulebooks/README.md`](../04-rulebooks/README.md). Once active, the
webhook endpoint is served by EDA Controller itself (not `localhost:5000`);
[`../06-running-the-demo.md`](../06-running-the-demo.md) covers finding its
URL and testing it with `curl`.

## `module_defaults` and the `ansible.controller` action group

`configure_controller.yml` sets `controller_host`, `controller_oauthtoken`,
and `validate_certs` once via `module_defaults: group/ansible.controller.controller`,
rather than repeating them on all eleven tasks. If your installed version of
the `ansible.controller` collection doesn't define this action group, every
task will fail with an authentication error - in that case, add the three
parameters directly to each `ansible.controller.*` task instead.

After this, continue to
[`../06-running-the-demo.md`](../06-running-the-demo.md) for the end-to-end
walkthrough.

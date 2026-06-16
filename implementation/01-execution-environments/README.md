# Execution & Decision Environments

Three container image definitions, built with `ansible-builder` and pushed
to Private Automation Hub (or any registry Cluster A's AAP can pull from).

| Directory | Image | Used by | Status |
|---|---|---|---|
| `ee-ocpvirt/` | `ee-ocpvirt` | `Backup VM - OpenShift Virtualization`, `Restore VM - OpenShift Virtualization` job templates | **Executed** |
| `ee-vmware/` | `ee-vmware` | `Backup VM - VMware`, `Restore VM - VMware` job templates | Reference only — not built/pushed |
| `de-self-heal/` | `de-self-heal` | The EDA rulebook activation in `04-rulebooks/self_heal_vm.yml` | **Executed** |

`Patch & Health-Check VM` deliberately uses AAP's built-in
**"Default execution environment"** — it only needs `ansible.builtin` and
`ansible.posix`, which that image already provides.

## Build and push `ee-ocpvirt`

```bash
cd 01-execution-environments/ee-ocpvirt
ansible-builder build \
  --tag <your-private-hub>/ee-ocpvirt:latest \
  --container-runtime podman

podman push <your-private-hub>/ee-ocpvirt:latest
```

## Build and push `de-self-heal`

```bash
cd 01-execution-environments/de-self-heal
ansible-builder build \
  --tag <your-private-hub>/de-self-heal:latest \
  --container-runtime podman

podman push <your-private-hub>/de-self-heal:latest
```

Both image references (`<your-private-hub>/ee-ocpvirt:latest` and
`<your-private-hub>/de-self-heal:latest`) are then used as
`ee_ocpvirt_image` and `de_self_heal_image` in
[`05-controller-as-code/vars/controller_objects.yml`](../05-controller-as-code/vars/controller_objects.yml),
which registers them with Automation Controller / EDA Controller.

## `ee-vmware` (not built)

`ee-vmware/` contains the same three files as `ee-ocpvirt/`, with
`community.vmware` and `pyvmomi` instead of `kubernetes.core` and
`kubernetes` — included so the VMware playbooks in `03-playbooks/` have a
documented runtime, but **not built or pushed**, since there's nothing to
run them against.

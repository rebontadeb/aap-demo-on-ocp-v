# EDA Rulebook

`self_heal_vm.yml` implements the use case from
[Chapter 8](../../chapters/08-eda-self-healing-usecase.md): a monitoring
alert fired **independently of any AAP job** triggers an automatic restore.

It runs inside the `de-self-heal` Decision Environment
(`01-execution-environments/de-self-heal/`), activated via
`05-controller-as-code/configure_eda.yml`.

## What it does

| | |
|---|---|
| Source | `ansible.eda.webhook` — listens for HTTP POST requests on port `5000` |
| Rule (executed) | If the posted JSON looks like a *firing* `VMServiceDown` alert with `platform: ocpvirt`, call `Restore VM - OpenShift Virtualization` with `limit` set to the VM named in the alert |
| Rule (reference only) | Same shape, for `platform: vmware` → `Restore VM - VMware` — not exercised, no VMware infra |

The condition is written to match the **labels** structure typical of a
Prometheus Alertmanager webhook payload — `status`, `labels.alertname`,
`labels.platform`, `labels.vm_name` — without requiring an actual
Alertmanager, so it can be tested with `curl`.

## Testing locally with `ansible-rulebook`

Before wiring this up as an EDA Controller activation, it can be run
standalone:

```bash
export CONTROLLER_HOST="https://aap-controller.apps.cluster-a.example.com"
export CONTROLLER_TOKEN="<automation controller API token>"

ansible-rulebook \
  --rulebook self_heal_vm.yml \
  -i ../02-inventory/hosts.yml \
  --print-events
```

Then, from another terminal, simulate the delayed failure described in
Chapter 8 — a VM whose health check passed during the workflow, but whose
critical service has since crashed:

```bash
curl -X POST http://localhost:5000/endpoint \
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

`--print-events` shows the incoming event; if the condition matches,
`ansible-rulebook` calls Automation Controller to launch
`Restore VM - OpenShift Virtualization` with `limit=rhel9-app-01` —
exactly as it would once activated inside EDA Controller.

## Moving to EDA Controller

`05-controller-as-code/configure_eda.yml` registers this rulebook (via an
EDA **Project** pointing at this Git repo), the `de-self-heal` **Decision
Environment**, and a **Rulebook Activation** that runs it continuously —
so the manual `ansible-rulebook` invocation above is for testing only.

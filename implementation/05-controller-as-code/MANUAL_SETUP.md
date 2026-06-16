# Manual AAP Setup (Alternative to configure_controller.yml)

Use this if `configure_controller.yml` cannot run against your AAP instance
(e.g. AAP 2.5 unified gateway with no Automation Hub access). All steps are
performed in the Automation Controller UI.

---

## 1. Organization

**Automation Controller → Access → Organizations → Add**

| Field | Value |
|---|---|
| Name | `VM Operations` |

---

## 2. Credentials

**Automation Controller → Resources → Credentials → Add**

### ocpvirt-api-token

| Field | Value |
|---|---|
| Name | `ocpvirt-api-token` |
| Organization | `VM Operations` |
| Credential Type | `OpenShift or Kubernetes API Bearer Token` |
| Host | Cluster B API URL (`oc whoami --show-server` on Cluster B) |
| API authentication bearer token | Output of `oc create token aap-vm-operator -n vm-workloads --duration=8760h` |
| Verify SSL | Unchecked (or checked if you have a valid cert) |

### vm-guest-ssh

| Field | Value |
|---|---|
| Name | `vm-guest-ssh` |
| Organization | `VM Operations` |
| Credential Type | `Machine` |
| Username | `cloud-user` |
| SSH Private Key | Paste the private key whose public half is in each VM's `~/.ssh/authorized_keys` |
| Privilege Escalation Method | `sudo` |

---

## 3. Execution Environment

**Automation Controller → Administration → Execution Environments → Add**

| Field | Value |
|---|---|
| Name | `ee-ocpvirt` |
| Image | `quay.io/rebontadeb/ansible/ee-ocpvirt` |
| Organization | `VM Operations` |

---

## 4. Project

**Automation Controller → Resources → Projects → Add**

| Field | Value |
|---|---|
| Name | `vm-lifecycle-automation` |
| Organization | `VM Operations` |
| Source Control Type | `Git` |
| Source Control URL | `https://github.com/rebontadeb/aap-demo-on-ocp-v.git` |
| Source Control Branch | `main` |
| Options | Check `Update Revision on Launch` |

Wait for the project to sync (status turns green) before continuing.

---

## 5. Inventory

**Automation Controller → Resources → Inventories → Add → Add Inventory**

| Field | Value |
|---|---|
| Name | `VM Fleet` |
| Organization | `VM Operations` |

After saving, open the inventory → **Sources** tab → **Add**:

| Field | Value |
|---|---|
| Name | `VM Fleet - from Git` |
| Source | `Sourced from a Project` |
| Project | `vm-lifecycle-automation` |
| Inventory file | `implementation/02-inventory/hosts.yml` |
| Options | Check `Overwrite` and `Update on Launch` |

Click **Save**, then **Sync** to pull the hosts in.

---

## 6. Job Templates

**Automation Controller → Resources → Templates → Add → Add Job Template**

Create all three below. For each, check **Prompt on launch** next to the
**Limit** field so a single VM can be targeted at launch time.

### Backup VM - OpenShift Virtualization

| Field | Value |
|---|---|
| Name | `Backup VM - OpenShift Virtualization` |
| Job Type | `Run` |
| Inventory | `VM Fleet` |
| Project | `vm-lifecycle-automation` |
| Playbook | `implementation/03-playbooks/backup_ocpvirt.yml` |
| Execution Environment | `ee-ocpvirt` |
| Credentials | `ocpvirt-api-token` |
| Limit | *(leave blank, check **Prompt on launch**)* |

### Patch & Health-Check VM

| Field | Value |
|---|---|
| Name | `Patch & Health-Check VM` |
| Job Type | `Run` |
| Inventory | `VM Fleet` |
| Project | `vm-lifecycle-automation` |
| Playbook | `implementation/03-playbooks/patch_and_healthcheck.yml` |
| Execution Environment | `Default execution environment` |
| Credentials | `vm-guest-ssh` |
| Limit | *(leave blank, check **Prompt on launch**)* |

### Restore VM - OpenShift Virtualization

| Field | Value |
|---|---|
| Name | `Restore VM - OpenShift Virtualization` |
| Job Type | `Run` |
| Inventory | `VM Fleet` |
| Project | `vm-lifecycle-automation` |
| Playbook | `implementation/03-playbooks/restore_ocpvirt.yml` |
| Execution Environment | `ee-ocpvirt` |
| Credentials | `ocpvirt-api-token` |
| Limit | *(leave blank, check **Prompt on launch**)* |

---

## 7. Workflow Job Template

**Automation Controller → Resources → Templates → Add → Add Workflow Template**

| Field | Value |
|---|---|
| Name | `VM Patch Cycle - OpenShift Virtualization` |
| Organization | `VM Operations` |
| Inventory | `VM Fleet` |
| Limit | *(leave blank, check **Prompt on launch**)* |

After saving, click **Visualizer** and build the chain:

```
[ Backup VM - OpenShift Virtualization ]
              |
         on SUCCESS
              |
              ▼
  [ Patch & Health-Check VM ]
              |
         on FAILURE
              |
              ▼
[ Restore VM - OpenShift Virtualization ]
```

Steps in the visualizer:
1. Click **Start** → select `Backup VM - OpenShift Virtualization` → Save
2. Hover the backup node → click **+** → select `On Success` → select `Patch & Health-Check VM` → Save
3. Hover the patch node → click **+** → select `On Failure` → select `Restore VM - OpenShift Virtualization` → Save

Click **Save** on the workflow template.

---

## What's next

With all objects created, continue to
[`../06-running-the-demo.md`](../06-running-the-demo.md) — **Step 5**
(launch the happy-path workflow run).

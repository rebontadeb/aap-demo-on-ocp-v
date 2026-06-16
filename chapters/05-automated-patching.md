# Chapter 5: Automating the Patch

With backups handled, this chapter automates the patch itself — and,
critically, the **health check** that decides whether the patch succeeded
well enough to leave the VM running, or whether it's time to roll back to
the snapshot from Chapter 4.

## One job template, any platform

Unlike backup and restore, **patching happens at the guest-OS level** —
over SSH, using `ansible.builtin` and `ansible.posix` modules that ship in
any default Execution Environment. It doesn't matter whether the VM lives
on OpenShift Virtualization or VMware; once SSH is reachable, the patch
steps are identical.

| Setting | Value |
|---|---|
| Name | `Patch & Health-Check VM` |
| Inventory | `VM Fleet` (targets hosts from *either* group — no platform-specific limit) |
| Project | `vm-lifecycle-automation` |
| Playbook | `playbooks/patch_and_healthcheck.yml` |
| Execution Environment | default minimal EE (no platform-specific collections needed) |
| Credentials | `vm-guest-ssh` |
| Survey | `critical_service` (default `httpd`), `health_check_port` (default `443`) |

The survey lets whoever launches the job (or the workflow in Chapter 6)
specify *what "healthy" means* for the VMs being patched, without editing
the playbook.

## The playbook

```yaml
# playbooks/patch_and_healthcheck.yml
---
- name: Patch VM and verify health
  hosts: "{{ target_hosts | default('all') }}"
  become: true
  vars:
    critical_service: "{{ health_check_service | default('httpd') }}"
    health_check_port: "{{ health_check_port | default(443) }}"

  tasks:
    - name: Patch all packages
      ansible.builtin.dnf:
        name: "*"
        state: latest
      register: patch_result

    - name: Reboot if the kernel or core libraries were updated
      ansible.builtin.reboot:
        reboot_timeout: 600
      when: patch_result.changed

    - name: Wait for the VM to come back online
      ansible.builtin.wait_for_connection:
        delay: 10
        timeout: 300

    - name: Check that the critical service is active
      ansible.builtin.systemd:
        name: "{{ critical_service }}"
        state: started
      check_mode: true
      register: service_status
      failed_when: false

    - name: Check that the application port responds
      ansible.builtin.wait_for:
        port: "{{ health_check_port }}"
        host: "{{ inventory_hostname }}"
        timeout: 30
        state: started
      register: port_check
      failed_when: false

    - name: Determine overall patch health
      ansible.builtin.set_fact:
        patch_healthy: >-
          {{ (service_status.status.ActiveState | default('') == 'active')
             and (port_check.state | default('') == 'started') }}

    - name: Record health-check result for the workflow
      ansible.builtin.set_stats:
        data:
          patch_healthy: "{{ patch_healthy }}"
        per_host: true

    - name: Fail this host if the health check did not pass
      ansible.builtin.fail:
        msg: >-
          Post-patch health check failed for {{ inventory_hostname }}:
          service '{{ critical_service }}' active={{ service_status.status.ActiveState | default('unknown') }},
          port {{ health_check_port }} state={{ port_check.state | default('unknown') }}
      when: not patch_healthy
```

## Two signals, one outcome

This playbook produces the health-check result **two ways on purpose**:

1. **`set_stats` (`patch_healthy`)** — a structured artifact that downstream
   workflow steps and (later) EDA can read directly.
2. **`fail`** — if the health check didn't pass, the task explicitly fails
   the play for that host. This is what makes Automation Controller mark
   the job as **failed**, which is the signal the workflow in Chapter 6
   uses to branch toward "restore."

In other words: a *healthy* VM finishes this job template with status
**successful**. An *unhealthy* VM finishes with status **failed** — on
purpose — because "failed" is exactly the trigger the next chapter is
waiting for.

## What happens next

Run on its own, this job template patches a VM and either succeeds or
fails based on the health check — but does nothing about a failure. That's
deliberate: **patching shouldn't also decide how to recover**. Recovery is
the job of the workflow, which is where Chapter 6 picks up.

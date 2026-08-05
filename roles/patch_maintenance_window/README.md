# patch_maintenance_window

**Category:** Patch Management
**Supports:** RHEL/CentOS/Rocky/Alma 8/9, Ubuntu 20.04/22.04/24.04, Debian 11/12
(OS-agnostic — no package/service management, works identically on every distro)

## Purpose

Orchestration guard role meant to be included at the **top** of other patch
playbooks. It evaluates the current date/time — always computed on the
**control node** via `delegate_to: localhost` so a mixed-timezone fleet
agrees on one consistent window — against a `maintenance_window` var (allowed
days, start/end time, IANA timezone, including windows that cross midnight),
and exposes a single boolean fact, `patch_maintenance_window_open`, that
downstream roles guard their tasks with. If outside the window, it either
prints a clear skip message (default) or fails the play (strict mode), and
supports an explicit `maintenance_window_force_run` override for emergency
out-of-window patching.

## Key variables

| Variable | Default | Description |
|---|---|---|
| `maintenance_window.enabled` | `true` | Master on/off switch |
| `maintenance_window.days` | `["Tue", "Thu"]` | Allowed 3-letter day abbreviations; `[]` = every day |
| `maintenance_window.start_time` / `.end_time` | `"22:00"` / `"02:00"` | HH:MM 24h window bounds; `end_time < start_time` means the window crosses midnight |
| `maintenance_window.timezone` | `"UTC"` | IANA timezone the window is interpreted in |
| `maintenance_window_force_run` | `false` | Bypass the window check entirely (e.g. `-e maintenance_window_force_run=true` for an emergency patch) |
| `maintenance_window_fail_instead_of_skip` | `false` | Fail the play instead of just skipping downstream tasks when outside the window |

## Example play

```yaml
- name: Patch fleet, but only inside the approved maintenance window
  hosts: "{{ target_hosts | default('all') }}"
  become: true
  gather_facts: true
  roles:
    - role: patch_maintenance_window
      vars:
        maintenance_window:
          enabled: true
          days: ["Sat", "Sun"]
          start_time: "01:00"
          end_time: "05:00"
          timezone: "America/Chicago"
    - role: os_patch_rolling_update
      when: patch_maintenance_window_open | bool
```

## Secrets manager pattern

This role does not require any secrets — it only evaluates date/time logic.
No Vault/AWS Secrets Manager lookups are used.

## Notes

- Always guard the roles that follow this one with
  `when: patch_maintenance_window_open | bool` — this role does not itself
  block or skip other roles' tasks; it only computes the fact.
- Because the check is delegated to `localhost` and uses `run_once: true`
  for the debug/fail messages, this role is intentionally cheap to include
  even on very large inventories — the date computation happens once per
  play, not once per host.
- For an emergency out-of-window patch, prefer `-e maintenance_window_force_run=true`
  over disabling `maintenance_window.enabled`, so the override is visible in
  the ad-hoc command / CI log rather than buried in group_vars.

## References

- [ansible.builtin.command module docs](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/command_module.html)
- [Ansible delegate_to documentation](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_delegation.html)
- [Jinja2 comparison expressions in Ansible conditionals](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_conditionals.html)

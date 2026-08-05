# password_policy_enforcement

**Category:** Security & Compliance
**Supports:** RHEL/CentOS/Rocky/Alma 8/9, Ubuntu 20.04/22.04/24.04, Debian 11/12

## Purpose

Enforces password complexity via PAM `pwquality` (minimum length, character-class
requirements, max repeated characters, difference-from-previous requirement), account aging
via `login.defs` (`PASS_MAX_DAYS`, `PASS_MIN_DAYS`, `PASS_WARN_AGE`), password history reuse
prevention via `pam_pwhistory`, and account lockout after repeated failed logins via
`pam_faillock` (preferred) or `pam_tally2` (fallback for older Debian releases).

## Key variables

| Variable | Default | Description |
|---|---|---|
| `password_min_length` | `14` | Minimum password length |
| `password_min_class` | `3` | Minimum distinct character classes required |
| `password_dcredit`/`ucredit`/`lcredit`/`ocredit` | `-1` each | Require at least one digit/upper/lower/special character |
| `password_maxrepeat` | `3` | Max identical consecutive characters allowed |
| `password_difok` | `8` | Minimum changed characters vs. previous password |
| `password_history_remember` | `5` | Number of previous passwords disallowed for reuse |
| `password_enforce_for_root` | `true` | Apply pwquality checks to root account too |
| `password_max_days` / `min_days` / `warn_age` | `90` / `7` / `14` | `login.defs` account aging policy |
| `password_lockout_mechanism` | `pam_faillock` | `pam_faillock` or `pam_tally2` |
| `password_lockout_attempts` | `5` | Failed attempts before lockout |
| `password_lockout_unlock_time` | `900` | Seconds before auto-unlock (`0` = manual admin unlock only) |
| `password_lockout_even_deny_root` | `false` | Also lock out the root account on repeated failures |

## Example play

```yaml
- hosts: linux_fleet
  become: true
  roles:
    - role: password_policy_enforcement
      vars:
        password_min_length: 16
        password_lockout_attempts: 3
        password_lockout_unlock_time: 1800
```

## Notes

- On RHEL 8/9 hosts using an `authselect` custom profile, prefer
  `authselect enable-feature with-faillock` plus `authselect apply-changes` for long-term
  consistency instead of relying solely on direct PAM file edits — this role edits the PAM
  files directly for broad compatibility and documents this caveat via an in-task notice.
- No service restart is required; PAM/login.defs changes take effect on the next new login
  or password-change attempt.

## References

- [pam_pwquality manual](https://man7.org/linux/man-pages/man8/pam_pwquality.8.html)
- [pam_faillock manual](https://man7.org/linux/man-pages/man8/pam_faillock.8.html)
- [login.defs manual](https://man7.org/linux/man-pages/man5/login.defs.5.html)
- [ansible.builtin.lineinfile module docs](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/lineinfile_module.html)

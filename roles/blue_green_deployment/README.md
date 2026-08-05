# blue_green_deployment

**Category:** Application Deployment
**Supports:** RHEL/CentOS/Rocky/Alma 8/9, Ubuntu 20.04/22.04/24.04, Debian 11/12
(supported transitively through whichever `app_deploy_role` and load-balancer role it
includes — this role itself has no OS-specific tasks)

## Purpose

Generic, reusable blue/green deployment orchestrator. Given two Ansible inventory groups
(`blue_group` / `green_group`) and a `blue_green_active_color` marker for whichever color is
currently live, this role: (1) determines the idle color, (2) dynamically includes any
other role in this collection named by `app_deploy_role` (e.g. `nodejs_app_deployment`,
`java_app_deployment`, `docker_engine_setup`) and runs it against every host in the idle
color's group, (3) HTTP health-checks each newly-deployed idle host, (4) pauses for operator
approval (unless `blue_green_require_manual_approval: false`), then (5) flips the load
balancer (HAProxy or nginx, selected by `blue_green_lb_type`) to point its upstream/backend
at the idle color, completing the swap. The previously-live color is left running and idle,
ready for the next release cycle or an emergency rollback (flip `blue_green_active_color`
back and re-run).

## Key variables

| Variable | Default | Description |
|---|---|---|
| `blue_group` / `green_group` | `"blue_group"` / `"green_group"` | Inventory group names for each color |
| `blue_green_active_color` | `"blue"` | Which color is currently live |
| `app_deploy_role` | `"nodejs_app_deployment"` | Any app-deployment role in this collection, included dynamically |
| `blue_green_lb_type` | `"haproxy"` | `"haproxy"` or `"nginx"` |
| `blue_green_lb_hosts` | `"load_balancers"` | Inventory group of LB hosts updated on flip |
| `blue_green_backend_name` | `"app_backend"` | Backend/upstream block name being flipped |
| `blue_green_health_check_path` | `"/health"` | Path used for the pre-flip health check |
| `blue_green_require_manual_approval` | `true` | Pause for operator confirmation before flipping live traffic |

## Example play

```yaml
- hosts: localhost
  connection: local
  gather_facts: false
  roles:
    - role: blue_green_deployment
      vars:
        blue_group: web_blue
        green_group: web_green
        blue_green_active_color: blue
        app_deploy_role: nodejs_app_deployment
        blue_green_lb_type: haproxy
        blue_green_lb_hosts: load_balancers
        blue_green_backend_name: checkout_backend
        blue_green_backend_port: 3000
```

## Notes

- This role is designed to run from a control-node-facing play (`hosts: localhost`) since
  it orchestrates other hosts via `delegate_to` and `include_role` rather than acting on
  `inventory_hostname` directly.
- After a successful flip, persist the new `blue_green_active_color` value back into
  `group_vars/all.yml` (or your CMDB/state store) so the next run starts from the correct
  baseline — this role does not write that value back automatically.
- Rollback is simply re-running the role with `blue_green_active_color` set back to the
  previous value; the previously-idle color has been left running and can immediately take
  traffic again after another health check + flip.

## References

- [ansible.builtin.include_role module docs](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/include_role_module.html)
- [ansible.builtin.uri module docs](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/uri_module.html)
- [HAProxy configuration manual](https://docs.haproxy.org/2.8/configuration.html)

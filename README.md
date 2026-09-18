# ansible_role_routeros_firewall

Manage MikroTik RouterOS 7.x firewall rules via the RouterOS API. Applies firewall configuration for raw, filter, NAT, and mangle tables using `community.routeros.api_modify` with `ensure_order` and `handle_absent_entries: remove`.

Rules are provided as flat ordered lists per table per IP version. The list order determines rule order on the device. Rules removed from the list are removed from the device on the next run.

IPv4 and IPv6 are handled independently.

All API passwords are **always prompted at runtime** and never stored in variables or config files.

## What it does

- Apply firewall rules to raw, filter, NAT, and mangle tables
- Enforce exact rule ordering via `ensure_order: true`
- Remove rules from the device that are no longer in the configuration
- Clean up unmanaged fields on existing rules via `handle_entries_content: remove_as_much_as_possible`
- Configure connection tracking settings
- Manage firewall address lists
- Handle IPv4 and IPv6 independently

## Requirements

| Name | Version |
|------|---------|
| ansible | >= 2.10 |
| community.routeros | >= 2.2.0 |

### Python libraries

| Name | Description |
|------|-------------|
| `librouteros` | Required by `community.routeros.api_modify` for RouterOS API communication |

## How it works

The role is a thin wrapper around `community.routeros.api_modify`. Each table (raw, filter, NAT, mangle) has its own task file. The role passes user-provided rule lists directly to `api_modify` with zero transformation.

Key `api_modify` parameters used:
- `ensure_order: true` — enforces the exact order from the list
- `handle_absent_entries: remove` — deletes rules on the device that are not in the list
- `handle_entries_content: remove_as_much_as_possible` — cleans up fields not specified in the rule

Chain separators (disabled marker rules for visual grouping) are provided by the user as regular rules in the list. The role does not inject or modify them.

## Host Variables

### Firewall constants (per host)

These are referenced by rule lists via Jinja2 variables:

```yaml
routeros_firewall_dns_server_ip: "10.0.0.53"
routeros_firewall_bastion_host_ip: "10.0.0.42"
routeros_firewall_mgmt_https_port: 1112
routeros_firewall_mgmt_ssh_port: 1113
routeros_firewall_mgmt_winbox_port: 1114
routeros_firewall_dns_webui_port: 9756
routeros_firewall_tcp_mgmt_used_ports: "{{ routeros_firewall_mgmt_https_port }},{{ routeros_firewall_mgmt_ssh_port }},{{ routeros_firewall_mgmt_winbox_port }}"
routeros_firewall_trusted_vlan_name: "VLAN10_TRUSTED"
routeros_firewall_servers_vlan_name: "VLAN20_SERVERS"
```

### Rule lists

Each table has an IPv4 and IPv6 variable. All default to empty lists (the role will not touch a table if its list is empty).

| Variable | API Path |
|----------|----------|
| `routeros_firewall_ipv4_raw_rules` | `ip firewall raw` |
| `routeros_firewall_ipv4_filter_rules` | `ip firewall filter` |
| `routeros_firewall_ipv4_nat_rules` | `ip firewall nat` |
| `routeros_firewall_ipv4_mangle_rules` | `ip firewall mangle` |
| `routeros_firewall_ipv6_raw_rules` | `ipv6 firewall raw` |
| `routeros_firewall_ipv6_filter_rules` | `ipv6 firewall filter` |
| `routeros_firewall_ipv6_nat_rules` | `ipv6 firewall nat` |
| `routeros_firewall_ipv6_mangle_rules` | `ipv6 firewall mangle` |

### Other variables

| Variable | API Path |
|----------|----------|
| `routeros_firewall_ipv4_connection_tracking` | `ip firewall connection tracking` |
| `routeros_firewall_ipv4_address_lists` | `ip firewall address-list` |
| `routeros_firewall_ipv6_address_lists` | `ipv6 firewall address-list` |

## Rule format

Each rule is a dict with RouterOS API field names (hyphens, not underscores):

```yaml
routeros_firewall_ipv4_filter_rules:
  # Chain separator (start)
  - action: "accept"
    chain: "input"
    comment: "\t\t\t==============================[ CHAIN: INPUT ]=============================="
    disabled: "yes"

  # Actual rules
  - action: "accept"
    chain: "input"
    comment: "FI_A_Est/Rel/Unt"
    connection-state: "established,related,untracked"
    packet-mark: "no-mark"
  - action: "drop"
    chain: "input"
    comment: "FI_D_Invalid"
    connection-state: "invalid"
    packet-mark: "no-mark"
  - action: "drop"
    chain: "input"
    comment: "FI_D_ALL:ImplicitDeny"
    log: "yes"
    packet-mark: "no-mark"

  # Chain separator (end)
  - action: "drop"
    chain: "input"
    comment: "\t\t\t==============================[ END CHAIN: INPUT ]=============================="
    disabled: "yes"
```

## Role Defaults (`defaults/main.yml`)

| Variable | Description | Default |
|----------|-------------|---------|
| `routeros_firewall_api_hostname` | RouterOS API hostname | `"{{ ansible_host }}"` |
| `routeros_firewall_api_username` | RouterOS API username | `"{{ ansible_user }}"` |
| `routeros_firewall_api_tls` | Use TLS for API connection | `true` |
| `routeros_firewall_api_port` | API port | `8729` |
| `routeros_firewall_api_validate_certs` | Validate TLS certificates | `false` |
| `routeros_firewall_api_validate_cert_hostname` | Validate certificate hostname | `false` |

## host_vars structure

Use Ansible's directory-based host_vars to split configuration into manageable files:

```
host_vars/router_hapac/
  bootstrap.yml
  firewall_vars.yml
  firewall_connection_tracking.yml
  firewall_address_lists.yml
  firewall_raw.yml
  firewall_filter.yml
  firewall_nat.yml
```

Ansible automatically merges all files in the directory. Each file defines its own variables.

## Usage

### Playbook

```yaml
---
- name: Configure RouterOS firewall
  hosts: router_hapac
  gather_facts: no
  become: no

  roles:
    - routeros_firewall
```

### Tags

| Tag | What runs |
|-----|-----------|
| `firewall` | Everything |
| `connection_tracking` | Connection tracking only |
| `address_lists` | Address lists only |
| `raw` | Raw table only |
| `filter` | Filter table only |
| `nat` | NAT table only |
| `mangle` | Mangle table only |

```bash
# Apply only filter rules
ansible-playbook -i inventory.ini firewall.yml --tags filter

# Apply everything
ansible-playbook -i inventory.ini firewall.yml
```

## Task Flow

1. **variables.yml** - Prompt for API password, validate all input variables are the correct types
2. **connection_tracking.yml** - Apply connection tracking settings
3. **address_lists.yml** - Apply IPv4/IPv6 address lists
4. **raw.yml** - Apply IPv4/IPv6 raw table rules
5. **filter.yml** - Apply IPv4/IPv6 filter table rules
6. **nat.yml** - Apply IPv4/IPv6 NAT table rules
7. **mangle.yml** - Apply IPv4/IPv6 mangle table rules

## Drawbacks and limitations vs Terraform RouterOS provider

### No atomic transactions

`api_modify` applies rules sequentially. If a failure occurs mid-way through a table, the device is left in a partial state with some rules applied and others not. The Terraform RouterOS provider has the same limitation at the individual resource level, but Terraform's dependency graph and state tracking make recovery more predictable.

### No plan/preview

Unlike `terraform plan`, there is no built-in way to preview exactly what will change before applying. Ansible's `--check` mode is supported by `api_modify` and shows what would change, but it is less detailed than Terraform's plan output.

### No persistent state file

Terraform tracks the current state of all managed resources in a state file. This enables drift detection without contacting the device. Ansible has no state file — it derives desired state from variables on each run and compares against the live device, which means drift detection requires actually running the playbook.

### Order sensitivity

Rule order is implicit from list position. In large rulesets, this makes it harder to review changes in diffs compared to Terraform's named resources where each rule has a unique resource identifier. Moving a rule in the list produces a large diff even if the rule itself is unchanged.

### api_modify path limitations

Not all RouterOS paths are supported by `community.routeros.api_modify`. If a path is unsupported, it requires an upstream pull request to the `community.routeros` collection. The Terraform RouterOS provider supports arbitrary paths via the `routeros_rest_api` resource.

### No cross-resource dependency references

Terraform can reference outputs from other resources (e.g., use an IP address from a created interface in a firewall rule). Ansible requires manual variable wiring between tasks and roles.

### Chain separator overhead

The disabled marker rules used for visual chain grouping consume router memory (minimal but nonzero). This is a cosmetic trade-off for human readability in the RouterOS terminal and Winbox. The Terraform provider does not need this pattern since rules are identified by resource names, not comments.

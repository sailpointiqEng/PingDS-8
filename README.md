# idp-playbooks

Playbook project for **Ping Directory Server 8** (config store). Contains playbooks, roles, files, and vars. **Inventory is managed in Ansible Tower** (separate project/source).

## Project layout

```
idp-playbooks/
├── playbooks/       # Playbooks (run from Tower)
├── roles/           # Roles
├── files/           # Static files (e.g. DS ZIP)
├── vars/            # Additional variables
├── group_vars/      # Default group vars (Tower can override)
│   └── all.yml
├── ansible.cfg
└── README.md
```

## Ansible Tower

- **Project**: Point Tower project at this repo (idp-playbooks).
- **Inventory**: Manage in Tower; define group `pingds_config` and hosts; set host vars (`server_id`, `hostname`) and group/vault vars (secrets, `deployment_id`).
- **Job template**: Project = this repo, playbook = `playbooks/<name>.yml`, inventory = your Tower inventory.

## Playbooks

| Job template       | Playbook                                |
|--------------------|-----------------------------------------|
| PingDS – Full      | `playbooks/site.yml`                    |
| PingDS – Host prep | `playbooks/host_prepare.yml`            |
| PingDS – Install   | `playbooks/pingds_install.yml`          |
| PingDS – Configure | `playbooks/pingds_configure.yml`       |
| PingDS – Verify    | `playbooks/pingds_replication_verify.yml` |
| PingDS – Validate  | `playbooks/pingds_post_deploy_validate.yml` or `validate_only.yml` |
| PingDS – Config only | `playbooks/config_only.yml`          |

## Roles

- **pingds_host_prepare** – Java, user/group, disk, firewall, time sync
- **pingds_install** – Unpack DS ZIP, setup am-config (skips if installed)
- **pingds_configure** – Start DS
- **pingds_replication_verify** – dsreplication status
- **pingds_post_deploy_validate** – Service, binds, base DN, AM config admin, replication

## Defaults (group_vars/all.yml)

Paths: `/opt/apps/opendj`, `/opt/apps`; ports; `bootstrap_replication_servers`. Override in Tower as needed.
========================

## Ansible Tower setup

### Job templates

Create one job template per playbook. Suggested set:

| Template name       | Playbook                              | Purpose |
|---------------------|----------------------------------------|--------|
| PingDS – Full       | `playbooks/site.yml`                   | Full run: prepare → install → configure → start → replication verify → validate |
| PingDS – Host prepare | `playbooks/host_prepare.yml`         | Host prep only (Java, user, paths) |
| PingDS – Install    | `playbooks/pingds_install.yml`         | Unzip only |
| PingDS – Configure  | `playbooks/pingds_configure.yml`      | Run setup (config store) only |
| PingDS – Start      | `playbooks/pingds_start.yml`           | Start DS only |
| PingDS – Config only | `playbooks/config_only.yml`           | Configure + start + replication verify |
| PingDS – Replication verify | `playbooks/pingds_replication_verify.yml` | Replication check |
| PingDS – Validate   | `playbooks/pingds_post_deploy_validate.yml` | Post-deploy validation |

### Inventory

- Use a group named **`pingds_config`** (all playbooks target this group).
- Add your DS hosts to that group.

### Variables

**Vault (secrets):** `deployment_id`, `deployment_id_password`, `root_user_password`, `monitor_user_password`, `am_config_admin_password`

**Group vars (e.g. for `pingds_config`):** `bootstrap_replication_servers` (list, e.g. `["example1:8989","example2:8989","example3:8989"]`). Optional: `ds_zip_name`, `ds_install_path`, `ds_install_base`, ports.

**Host vars (per host):** `server_id` (e.g. 1, 2, 3), `hostname` (e.g. example1).

**Required for Configure (and Full / Config only):** All vault vars; group `bootstrap_replication_servers`; each host `server_id` and `hostname`.

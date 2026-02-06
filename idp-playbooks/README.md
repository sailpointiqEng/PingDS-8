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

## Ansible Tower setup

### Job templates

Create one job template per playbook. Point each template at this project, the playbook path below, and your inventory (group `pingds_config`). Attach a **Vault** credential for secrets (see Variables).

| Template name            | Playbook                                    | Purpose |
|--------------------------|---------------------------------------------|---------|
| **PingDS – Full**        | `playbooks/site.yml`                        | Full run: prepare → install → configure → start → replication verify → validate |
| **PingDS – Host prepare**| `playbooks/host_prepare.yml`                | Host prep only (JDK 17, opendj user/group, /opt/apps) |
| **PingDS – Install**     | `playbooks/pingds_install.yml`               | Unzip DS only (does not run setup) |
| **PingDS – Configure**  | `playbooks/pingds_configure.yml`            | Run setup as config store only (does not start server) |
| **PingDS – Start**       | `playbooks/pingds_start.yml`                 | Start DS only |
| **PingDS – Config only** | `playbooks/config_only.yml`                 | Configure + start + replication verify (no install) |
| **PingDS – Replication verify** | `playbooks/pingds_replication_verify.yml` | Replication status check |
| **PingDS – Validate**    | `playbooks/pingds_post_deploy_validate.yml`  | Post-deploy validation (service, binds, base DN, AM config admin, replication) |
| **PingDS – Validate only** | `playbooks/validate_only.yml`              | Same as Validate (optional second template) |

### Inventory

- Define a group named **`pingds_config`** (all playbooks use `hosts: pingds_config`).
- Add your DS hosts to that group (e.g. example1, example2, example3).
- Set **host vars** per host: `server_id`, `hostname`.

### Variables

**Vault (secrets)** – store in a Tower Vault credential or encrypted group/host vars:

| Variable                     | Required for |
|-----------------------------|--------------|
| `deployment_id`             | Configure, Full, Config only |
| `deployment_id_password`    | Configure, Full, Config only |
| `ds_rootUserPassword`        | Configure, Replication verify, Validate |
| `monitor_user_password`     | Configure, Full, Config only |
| `ds_am_ConfigAdminPassword`  | Configure, Validate |

**Group vars** (e.g. for group `pingds_config`):

| Variable                       | Required | Description |
|-------------------------------|----------|-------------|
| `bootstrap_replication_servers` | Yes (for configure) | List, e.g. `["example1:8989","example2:8989","example3:8989"]` |
| `ds_zip_name`                 | Optional | If install copies ZIP from project (e.g. `DS-8.0.2.zip`) |
| `ds_install_path`             | Optional | Default `"/opt/apps/opendj"` |
| `ds_install_base`             | Optional | Default `"/opt/apps"` |
| Ports, `root_user_dn`, `base_dn` | Optional | Override only if not using defaults |

**Host vars** (per host in `pingds_config`):

| Variable   | Required   | Description |
|------------|------------|-------------|
| `server_id` | Yes (for configure) | Unique per server, e.g. `1`, `2`, `3` |
| `hostname`  | Yes (for configure) | This host’s FQDN or name (e.g. `example1`) |

**Quick reference – which template needs what:**

- **Full / Config only / Configure:** All Vault vars; group `bootstrap_replication_servers`; each host `server_id`, `hostname`.
- **Host prepare / Install / Start:** No required vars (defaults OK); optional `ds_zip_name` for Install if using project file.
- **Replication verify:** `ds_rootUserPassword` (e.g. in Vault).
- **Validate:** `ds_rootUserPassword`, `ds_am_ConfigAdminPassword` (e.g. in Vault).

## Playbooks (summary)

- **site.yml** – Full deployment (all phases).
- **host_prepare.yml** – Host preparation only.
- **pingds_install.yml** – Unzip only.
- **pingds_configure.yml** – Run setup only.
- **pingds_start.yml** – Start server only.
- **config_only.yml** – Configure + start + replication verify.
- **pingds_replication_verify.yml** – Replication check.
- **pingds_post_deploy_validate.yml** / **validate_only.yml** – Post-deploy validation.

## Roles

| Role                      | Purpose |
|---------------------------|---------|
| **pingds_host_prepare**   | JDK 17 (verify/install), group/user opendj (no login), /opt/apps and /opt/apps/opendj, firewall, NTP |
| **pingds_install**        | Unzip DS distribution only; set ownership to opendj (no setup) |
| **pingds_configure**      | Run setup as DS config store (am-config + replication bootstrap); runs as opendj |
| **pingds_start**          | Ensure DS is running (start if not); runs as opendj |
| **pingds_replication_verify** | dsreplication status |
| **pingds_post_deploy_validate** | Service, root bind, base DN, AM config admin bind, replication status |

## Logging and step-by-step output (Ansible Tower)

Job output is tuned for clear step-by-step logs:

- **ansible.cfg** – `stdout_callback = yaml` so each task result is shown in a readable way; `display_ok_hosts` and `display_skipped_hosts` so all task outcomes appear in the job log.
- **Phase headers** – Each role starts with a `LOG |` debug task that prints the current phase and the steps that will run (e.g. "Phase: Host prepare. Steps: gather facts, disk check, JDK 17...").
- **Full deployment** – `site.yml` runs a single "Full deployment starting" log line before any role.

**More detail per task (module arguments, return values):** In the Tower Job Template, set **Verbosity** to **2** or **3**. Verbosity 0 (default) shows task names and results; 2–3 adds module input/output so you can see exactly what each task did.

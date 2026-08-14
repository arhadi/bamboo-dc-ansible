# Bamboo DC Automation

Ansible automation for deploying **Atlassian Bamboo Data Center** on RHEL 9, including infrastructure installation, PostgreSQL integration, JVM/systemd service management, cluster preparation, validation, and uninstallation.

## Repository Structure

```
bamboo-dc-ansible
├── ansible.cfg
├── inventory
│   ├── group_vars
│   │   └── bamboo.yml             # Variable overrides for the bamboo group
│   └── hosts.yml                  # Inventory of Bamboo nodes
├── playbooks
│   ├── install_bamboo.yml         # Full infrastructure install
│   └── uninstall_bamboo.yml       # Removes Bamboo installation
└── roles
    └── bamboo
        ├── defaults/main.yml      # Default role variables
        ├── files/                 # Installer archive (atlassian-bamboo-<version>.tar.gz)
        ├── handlers/main.yml      # Restart / reload handlers
        ├── meta/main.yml          # Role metadata
        ├── tasks/                 # Task files (see below)
        ├── templates/             # Jinja2 templates for config files
        └── vars/main.yml          # Role variables
```

## What This Automation Does

The `bamboo` role is broken into discrete task files, orchestrated by `roles/bamboo/tasks/main.yml`:

| Task file | Purpose |
|---|---|
| `precheck.yml` | Validates OS (RHEL 9, x86_64 only), checks minimum memory (4096 MB) and disk space (10 GB) on the base directory, verifies the installer archive exists and its `tar` integrity, checks PostgreSQL connectivity, warns on an HTTP port conflict, and sanity-checks required variables |
| `prerequisites.yml` | Installs required packages (Java 21, PostgreSQL 17 client, etc.), creates the Bamboo OS user/group, creates base/home directories, validates Java 21, resolves and validates the actual `JAVA_HOME`, and validates the PostgreSQL 17 client |
| `install.yml` | Extracts the Bamboo archive (skipped if already installed), renames the extracted directory into the target install directory, validates the start/stop scripts, sets ownership, and makes the scripts executable |
| `database.yml` | Validates PostgreSQL login, database encoding (UTF8), database ownership, and that the PostgreSQL major version matches `postgres_version` |
| `configure.yml` | Creates `BAMBOO_HOME` and its `logs`/`temp` subdirectories, deploys `bamboo-init.properties`, and sets JVM min/max memory and `BAMBOO_HOME` in `setenv.sh` |
| `cluster.yml` | Displays Bamboo Data Center deployment-mode information and validates/re-asserts ownership on `BAMBOO_HOME` — runs only when `bamboo_clustered` is `true`; actual cluster initialization is completed by the Bamboo Setup Wizard |
| `systemd.yml` | Deploys the systemd unit, reloads the daemon, and enables/starts the Bamboo service |
| `validate.yml` | Confirms install/home directories and the systemd service exist and are active/enabled, waits for and checks the HTTP endpoint, and validates the configured JVM memory settings |
| `uninstall.yml` | Stops/disables the service, removes the systemd unit, removes the installation directory (by default), and optionally purges `BAMBOO_HOME`, the OS user, and the OS group (PostgreSQL is never touched) |

## Playbooks

### `playbooks/install_bamboo.yml`
Runs the full role (`precheck` → `prerequisites` → `install` → `database` → `configure` → `cluster` [if clustered] → `systemd` → `validate`). Use this for the initial infrastructure setup.

```bash
ansible-playbook -i inventory/hosts.yml playbooks/install_bamboo.yml
```

### `playbooks/uninstall_bamboo.yml`
Removes the Bamboo installation via `tasks/uninstall.yml`. By default this removes the installation directory but preserves `BAMBOO_HOME`, the OS user, and the OS group. Pass the relevant `bamboo_purge_data`/`bamboo_remove_*` overrides to remove more.

```bash
ansible-playbook -i inventory/hosts.yml playbooks/uninstall_bamboo.yml

# Full purge (also removes BAMBOO_HOME, OS user, and OS group):
ansible-playbook -i inventory/hosts.yml playbooks/uninstall_bamboo.yml \
  -e bamboo_purge_data=true -e bamboo_remove_home=true \
  -e bamboo_remove_user=true -e bamboo_remove_group=true
```

## End-to-End Deployment Flow

1. Populate `inventory/hosts.yml` and `inventory/group_vars/bamboo.yml` with your environment's values.
2. Place the Bamboo installer archive (e.g. `atlassian-bamboo-<version>.tar.gz`) in `roles/bamboo/files/`.
3. Ensure an external PostgreSQL 17 instance is reachable, with the target database/user already created.
4. Run `install_bamboo.yml` to provision the OS user, install Bamboo, configure `bamboo-init.properties`/JVM memory, and start the service.
5. Open `http://<host>:<bamboo_http_port>` in a browser, complete the **Bamboo Setup Wizard**, and apply the Data Center license.
6. Re-run validation (via `install_bamboo.yml`, or by re-invoking the role's `validate` tasks) at any point to confirm service health.

## Key Variables

These are referenced throughout the role and should be defined in `roles/bamboo/defaults/main.yml`, `roles/bamboo/vars/main.yml`, or overridden in `inventory/group_vars/bamboo.yml`:

| Variable | Description |
|---|---|
| `bamboo_version` | Bamboo version being installed |
| `bamboo_archive` | Filename of the installer `.tar.gz` under `roles/bamboo/files/` |
| `bamboo_base_dir` | Base directory the archive is extracted into and used for the disk-space check |
| `bamboo_install_dir` | Final Bamboo installation directory |
| `bamboo_home` | `BAMBOO_HOME` directory (logs, temp) |
| `bamboo_user` / `bamboo_group` | OS user/group Bamboo runs as |
| `bamboo_java_package` | Java package installed via `dnf` (OpenJDK 21) |
| `bamboo_java_home` | Resolved `JAVA_HOME`, set automatically in `prerequisites.yml` |
| `bamboo_jvm_minimum_memory` / `bamboo_jvm_maximum_memory` | JVM `Xms`/`Xmx` values written to `setenv.sh` |
| `bamboo_http_port` | HTTP port Bamboo listens on |
| `bamboo_service_name` | Name of the systemd service |
| `bamboo_clustered` | Whether to run cluster preparation tasks |
| `bamboo_purge_data` | Master switch enabling optional removals during uninstall |
| `bamboo_remove_install_dir` | Whether uninstall removes the install directory (default `true`) |
| `bamboo_remove_home` / `bamboo_remove_user` / `bamboo_remove_group` | Fine-grained uninstall switches, only honored when `bamboo_purge_data` is also `true` |
| `postgres_host` / `postgres_port` / `postgres_database` / `postgres_user` / `postgres_password` | PostgreSQL connection details |
| `postgres_version` | Expected PostgreSQL major version, checked against the live server in `database.yml` |

## Requirements

- **Target OS:** RHEL 9 on x86_64 (enforced by `assert` checks in `precheck.yml`; the playbooks will fail on any other distribution/architecture)
- **Memory:** At least 4096 MB total system memory (enforced in `precheck.yml`)
- **Java:** OpenJDK 21 is installed and validated automatically during `prerequisites.yml`, and `JAVA_HOME` is resolved dynamically from `/usr/bin/java`
- **PostgreSQL:** An external, reachable PostgreSQL 17 instance with UTF8 encoding (login, encoding, ownership, and major version are checked in `database.yml`); the PostgreSQL 17 client package is installed and validated on the target host
- **Ansible control node:** `ansible-core` (adjust collection requirements per your Ansible version)
- **Files:** The Bamboo installer archive must be placed under `roles/bamboo/files/` before running the install playbook; `precheck.yml` validates both its presence and `tar` integrity

## Handlers

Defined in `roles/bamboo/handlers/main.yml`:

- **Reload systemd** — runs `systemd daemon_reload`
- **Restart Bamboo** — restarts the Bamboo systemd service

## Notes

- `install.yml` extracts the archive with `remote_src: true`, meaning the archive is expected to already exist at `{{ role_path }}/files/{{ bamboo_archive }}` on the path Ansible resolves on the control/target side per your setup — double-check this against how `bamboo_archive` is delivered in your environment.
- `database.yml` strictly enforces **UTF8 encoding** and checks that the PostgreSQL server's major version matches `postgres_version`, rather than hardcoding a specific PostgreSQL version.
- `cluster.yml` only displays deployment-mode information and validates/owns `BAMBOO_HOME`; actual Data Center cluster initialization is completed through the Bamboo Setup Wizard, not by this automation.
- `uninstall.yml` removes the installation directory by default (`bamboo_remove_install_dir: true`), but `BAMBOO_HOME`, the OS user, and the OS group are preserved unless both `bamboo_purge_data` and the corresponding `bamboo_remove_*` flag are set to `true`.
- PostgreSQL database/user lifecycle is never touched by this role, in installation or uninstallation.

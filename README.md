# Bamboo Data Center Ansible Automation

Production-oriented Ansible automation for installing, configuring, validating, and uninstalling **Atlassian Bamboo Data Center 12.1.10** on RHEL 9 with PostgreSQL 17.

This repository uses the Atlassian **`.tar.gz` archive distribution**. Bamboo does not use the dual archive/binary-installer workflow implemented in the Jira automation; the current Bamboo role is intentionally archive-only.

> Current tested/configured Bamboo version: **12.1.10**

## Contents

- [Overview](#overview)
- [Key Features](#key-features)
- [Directory Structure](#directory-structure)
- [Requirements](#requirements)
- [Installation Media](#installation-media)
- [Inventory](#inventory)
- [Configuration Variables](#configuration-variables)
- [Usage](#usage)
  - [Syntax Check](#syntax-check)
  - [Precheck](#precheck)
  - [Install Bamboo](#install-bamboo)
  - [Validate Bamboo](#validate-bamboo)
  - [Uninstall Bamboo](#uninstall-bamboo)
- [Role Task Flow](#role-task-flow)
- [Installation Behavior](#installation-behavior)
- [Bamboo Home Configuration](#bamboo-home-configuration)
- [Database Validation Behavior](#database-validation-behavior)
- [JVM Configuration](#jvm-configuration)
- [Systemd Ownership Model](#systemd-ownership-model)
- [Data Center Mode](#data-center-mode)
- [Idempotency](#idempotency)
- [Uninstall Behavior](#uninstall-behavior)
- [Validation](#validation)
- [Troubleshooting](#troubleshooting)
- [Security Recommendations](#security-recommendations)
- [Operational Commands](#operational-commands)

## Overview

The `bamboo` role manages the Bamboo infrastructure lifecycle:

```text
precheck
   |
   v
prerequisites
   |
   v
install
   |
   v
database validation
   |
   v
configure
   |
   v
cluster preparation (when enabled)
   |
   v
systemd
   |
   v
validate
```

The current automation installs Bamboo from:

```text
roles/bamboo/files/atlassian-bamboo-12.1.10.tar.gz
```

The role prepares the operating system, Bamboo installation and home directories, validates PostgreSQL, configures the Bamboo home pointer and JVM memory, installs an Ansible-managed systemd service, starts Bamboo, and validates the resulting deployment.

The first-run Bamboo application configuration is intentionally completed through the Bamboo Setup Wizard.

## Key Features

- RHEL 9 validation.
- x86_64 architecture validation.
- Minimum 4 GB system-memory validation.
- Minimum 10 GB free-space validation on `/app`.
- Bamboo 12.1.10 archive installation.
- Archive presence and `tar.gz` integrity validation.
- Java 21 installation and validation.
- Dynamic `JAVA_HOME` resolution.
- PostgreSQL 17 client installation and validation.
- PostgreSQL TCP connectivity validation.
- PostgreSQL authentication validation.
- PostgreSQL UTF8 encoding validation.
- PostgreSQL database-owner validation.
- PostgreSQL major-version validation.
- Dedicated Bamboo OS user and group.
- Custom installation directory under `/app`.
- Dedicated Bamboo home under `/app/bamboo-data`.
- JVM minimum and maximum heap configuration.
- Ansible-managed systemd service.
- HTTP endpoint validation.
- Data Center deployment preparation.
- Safe uninstall preserving application data and PostgreSQL by default.

## Directory Structure

```text
bamboo-dc-ansible/
├── ansible.cfg
├── inventory
│   ├── group_vars
│   │   └── bamboo.yml
│   └── hosts.yml
├── playbooks
│   ├── install_bamboo.yml
│   └── uninstall_bamboo.yml
└── roles
    └── bamboo
        ├── defaults
        │   └── main.yml
        ├── files
        │   └── atlassian-bamboo-12.1.10.tar.gz
        ├── handlers
        │   └── main.yml
        ├── meta
        │   └── main.yml
        ├── tasks
        │   ├── cluster.yml
        │   ├── configure.yml
        │   ├── database.yml
        │   ├── install.yml
        │   ├── main.yml
        │   ├── precheck.yml
        │   ├── prerequisites.yml
        │   ├── systemd.yml
        │   ├── uninstall.yml
        │   └── validate.yml
        └── templates
            ├── bamboo-init.properties.j2
            └── bamboo.service.j2
```

## Requirements

### Control/managed host

The supplied inventory uses local execution:

```yaml
---
all:
  children:
    bamboo:
      hosts:
        localhost:
          ansible_connection: local
```

The current implementation therefore expects the repository and Bamboo archive to be present on the target host where Ansible is executed.

### Operating system

The precheck enforces:

```text
RHEL-compatible OS family: RedHat
Major version:             9
Architecture:              x86_64
```

### Memory

The role requires at least:

```text
4096 MB
```

of total system memory.

### Disk

The role requires at least:

```text
10 GB
```

available on:

```text
/app
```

### Java

The configured package is:

```yaml
bamboo_java_package: java-21-openjdk
```

The role verifies that Java 21 is active and dynamically resolves the effective `JAVA_HOME` from `/usr/bin/java`.

### PostgreSQL

The current environment is configured for:

```yaml
postgres_version: 17
postgres_host: "127.0.0.1"
postgres_port: 15432
postgres_database: bamboo
postgres_user: bamboo
```

The database and database user must already exist. Database lifecycle is intentionally outside this role.

For production, do not leave:

```yaml
postgres_password: "bamboo"
```

in plaintext. Move the password to Ansible Vault.

## Installation Media

The current repository contains:

```text
roles/bamboo/files/atlassian-bamboo-12.1.10.tar.gz
```

Current observed file size:

```text
approximately 455 MB
```

Current SHA256 from the supplied environment:

```text
598937ddac7a781a42a6caf75e564f7cd1a489686e9e4a97415ac2cd4109dda4
```

Verify it with:

```bash
sha256sum roles/bamboo/files/atlassian-bamboo-12.1.10.tar.gz
```

The current precheck validates archive existence, non-zero size, and archive integrity with:

```bash
tar -tzf
```

It does not currently enforce the SHA256 value as an Ansible variable.

## Inventory

Current `inventory/hosts.yml`:

```yaml
---
all:
  children:
    bamboo:
      hosts:
        localhost:
          ansible_connection: local
```

For a remote deployment, adjust the inventory appropriately, for example:

```yaml
---
all:
  children:
    bamboo:
      hosts:
        bamboo01.example.com:
          ansible_host: 10.10.10.20
          ansible_user: ansible
```

Review `remote_src` behavior in `install.yml` before converting the current local-execution design to a remote control-node model.

## Configuration Variables

The primary environment overrides are stored in:

```text
inventory/group_vars/bamboo.yml
```

### Product

```yaml
bamboo_version: "12.1.10"

bamboo_user: bamboo
bamboo_group: bamboo
```

### Installation

```yaml
bamboo_base_dir: /app
bamboo_install_dir: "/app/bamboo-{{ bamboo_version }}"
bamboo_home: /app/bamboo-data

bamboo_archive: "atlassian-bamboo-{{ bamboo_version }}.tar.gz"
```

For the current version these resolve to:

```text
Install directory : /app/bamboo-12.1.10
Bamboo home       : /app/bamboo-data
Archive           : atlassian-bamboo-12.1.10.tar.gz
```

### Java

```yaml
bamboo_java_package: java-21-openjdk
bamboo_java_home: /usr/lib/jvm/java-21-openjdk-21.0.12.0.8-1.2.el9.x86_64
```

The configured value is subsequently replaced during role execution by the dynamically resolved Java home.

### Network

```yaml
bamboo_http_port: 8085
```

### PostgreSQL

```yaml
postgres_version: 17

postgres_host: "127.0.0.1"
postgres_port: 15432

postgres_database: bamboo
postgres_user: bamboo
postgres_password: "bamboo"
```

### JVM

```yaml
bamboo_jvm_minimum_memory: 2g
bamboo_jvm_maximum_memory: 4g
```

### Data Center

```yaml
bamboo_clustered: true
```

This enables the role's Data Center preparation task. Actual Bamboo application cluster initialization is not performed by `cluster.yml`.

### Service

```yaml
bamboo_service_name: bamboo
```

### Validation

```yaml
bamboo_validation_url: "http://127.0.0.1:8085"
bamboo_validation_retries: 60
bamboo_validation_delay: 10
```

Note that the current `validate.yml` directly uses the configured HTTP port and its own retry values for the endpoint check.

### Uninstall

```yaml
bamboo_remove_install_dir: true
bamboo_purge_data: false

bamboo_remove_home: false
bamboo_remove_user: false
bamboo_remove_group: false
```

The default uninstall therefore removes the Bamboo application binaries and service while preserving Bamboo Home and the Bamboo service account.

## Usage

Run all commands from the repository root:

```bash
cd /home/ansible/bamboo-dc-ansible
```

### Syntax Check

Installation:

```bash
ansible-playbook   -i inventory/hosts.yml   playbooks/install_bamboo.yml   --syntax-check
```

Uninstall:

```bash
ansible-playbook   -i inventory/hosts.yml   playbooks/uninstall_bamboo.yml   --syntax-check
```

### Precheck

The current `main.yml` uses `include_tasks` without tags. To run only precheck independently, invoke the role task file from a temporary/precheck playbook or add role tags before relying on `--tags precheck`.

During a normal installation the precheck automatically runs first and validates:

- operating system;
- architecture;
- memory;
- `/app` existence;
- available disk space;
- Bamboo archive existence;
- Bamboo archive integrity;
- existing installation state;
- PostgreSQL TCP reachability;
- HTTP port state;
- required configuration variables.

### Install Bamboo

Run:

```bash
ansible-playbook   -i inventory/hosts.yml   playbooks/install_bamboo.yml
```

The playbook executes:

```text
precheck
prerequisites
install
database
configure
cluster (when bamboo_clustered=true)
systemd
validate
```

After infrastructure installation, open:

```text
http://<BAMBOO_SERVER_IP>:8085
```

and complete the Bamboo Setup Wizard and Data Center licensing.

### Validate Bamboo

The current repository does not contain a separate `validate_bamboo.yml` playbook.

Validation is automatically executed at the end of:

```text
playbooks/install_bamboo.yml
```

For manual checks, see [Operational Commands](#operational-commands).

### Uninstall Bamboo

Default safe uninstall:

```bash
ansible-playbook   -i inventory/hosts.yml   playbooks/uninstall_bamboo.yml
```

Default behavior:

```text
Bamboo service      -> stopped/removed
Installation        -> removed
BAMBOO_HOME         -> preserved
Bamboo OS user      -> preserved
Bamboo OS group     -> preserved
PostgreSQL database -> preserved
PostgreSQL user     -> preserved
```

For an explicit filesystem/account purge:

```bash
ansible-playbook   -i inventory/hosts.yml   playbooks/uninstall_bamboo.yml   -e bamboo_purge_data=true   -e bamboo_remove_home=true   -e bamboo_remove_user=true   -e bamboo_remove_group=true
```

The PostgreSQL database and database user remain untouched.

## Role Task Flow

`roles/bamboo/tasks/main.yml` executes:

| Order | Task file | Purpose |
|---:|---|---|
| 1 | `precheck.yml` | Platform, capacity, archive, PostgreSQL and port checks |
| 2 | `prerequisites.yml` | Packages, service account, directories, Java and PostgreSQL client |
| 3 | `install.yml` | Extract and normalize Bamboo archive installation |
| 4 | `database.yml` | Validate PostgreSQL login, encoding, owner and version |
| 5 | `configure.yml` | Configure Bamboo home pointer and JVM settings |
| 6 | `cluster.yml` | Data Center preparation when `bamboo_clustered=true` |
| 7 | `systemd.yml` | Install, enable and start the Bamboo service |
| 8 | `validate.yml` | Validate filesystem, service, HTTP and JVM state |

The uninstall playbook directly includes:

```text
roles/bamboo/tasks/uninstall.yml
```

## Installation Behavior

`install.yml` first determines whether Bamboo is already installed by checking:

```text
{{ bamboo_install_dir }}
{{ bamboo_install_dir }}/bin/start-bamboo.sh
```

If both exist, extraction is skipped.

For a new installation, Ansible extracts:

```text
roles/bamboo/files/atlassian-bamboo-12.1.10.tar.gz
```

into:

```text
/app
```

The Atlassian archive directory:

```text
/app/atlassian-bamboo-12.1.10
```

is then renamed to:

```text
/app/bamboo-12.1.10
```

The role validates:

```text
/app/bamboo-12.1.10/bin/start-bamboo.sh
/app/bamboo-12.1.10/bin/stop-bamboo.sh
```

and normalizes ownership to:

```text
bamboo:bamboo
```

The start and stop scripts are set executable with mode:

```text
0750
```

## Bamboo Home Configuration

The role configures Bamboo Home through:

```text
/app/bamboo-12.1.10/atlassian-bamboo/WEB-INF/classes/bamboo-init.properties
```

using:

```text
roles/bamboo/templates/bamboo-init.properties.j2
```

Template:

```properties
# ============================================================
# Atlassian Bamboo
# Managed by Ansible
#
# DO NOT EDIT MANUALLY
# ============================================================

bamboo.home={{ bamboo_home }}
```

For the current inventory this resolves to:

```properties
bamboo.home=/app/bamboo-data
```

The role also creates:

```text
/app/bamboo-data
/app/bamboo-data/logs
/app/bamboo-data/temp
```

owned by:

```text
bamboo:bamboo
```

## Database Validation Behavior

The current Bamboo automation validates the database before starting Bamboo.

It does **not** create the PostgreSQL database or role.

The database is expected to have been provisioned separately, for example by the PostgreSQL Ansible automation.

### Connectivity

The role first waits for:

```text
127.0.0.1:15432
```

### Authentication

It executes an equivalent of:

```bash
PGPASSWORD='<password>' psql   -h 127.0.0.1   -p 15432   -U bamboo   -d bamboo   -tAc   "SELECT current_database() || '|' || current_user;"
```

Expected result:

```text
bamboo|bamboo
```

### Encoding

The role requires:

```text
UTF8
```

### Ownership

The role requires the configured database owner to match:

```text
bamboo
```

### PostgreSQL version

The configured major version is:

```yaml
postgres_version: 17
```

The role compares this with the live PostgreSQL server's reported major version.

## JVM Configuration

The role validates:

```text
/app/bamboo-12.1.10/bin/setenv.sh
```

and modifies the Bamboo JVM memory defaults.

Configured values:

```yaml
bamboo_jvm_minimum_memory: 2g
bamboo_jvm_maximum_memory: 4g
```

The expected managed lines are:

```bash
: ${JVM_MINIMUM_MEMORY:=2g}
: ${JVM_MAXIMUM_MEMORY:=4g}
```

The systemd unit also exports the corresponding environment variables.

After installation, inspect the running Java command with:

```bash
ps -ef | grep '[j]ava'
```

and inspect `setenv.sh` with:

```bash
grep -E   'JVM_MINIMUM_MEMORY|JVM_MAXIMUM_MEMORY'   /app/bamboo-12.1.10/bin/setenv.sh
```

## Systemd Ownership Model

Bamboo service lifecycle is managed by Ansible.

Template:

```text
roles/bamboo/templates/bamboo.service.j2
```

Installed unit:

```text
/etc/systemd/system/bamboo.service
```

Important properties include:

```ini
[Service]
Type=forking

User=bamboo
Group=bamboo

Environment="JAVA_HOME=<resolved-java-home>"
Environment="JVM_MINIMUM_MEMORY=2g"
Environment="JVM_MAXIMUM_MEMORY=4g"

UMask=0027

ExecStart=/app/bamboo-12.1.10/bin/start-bamboo.sh
ExecStop=/app/bamboo-12.1.10/bin/stop-bamboo.sh

TimeoutStartSec=600
TimeoutStopSec=300

Restart=no

LimitNOFILE=65536
```

The service is enabled and started by `systemd.yml`.

Useful commands:

```bash
systemctl status bamboo --no-pager -l
systemctl is-enabled bamboo
systemctl is-active bamboo
journalctl -u bamboo -n 100 --no-pager
```

## Data Center Mode

The current inventory enables:

```yaml
bamboo_clustered: true
```

This causes:

```text
roles/bamboo/tasks/cluster.yml
```

to run.

The current cluster task performs **Data Center preparation only**. It:

- reports the deployment mode;
- verifies Bamboo Home exists;
- validates Bamboo Home is a directory;
- reasserts Bamboo Home ownership and permissions;
- reports that application cluster initialization must be completed by Bamboo setup.

It does not currently configure a shared filesystem, node ID, Hazelcast/cluster networking, or other multi-node application-level settings.

The role therefore distinguishes:

```text
Infrastructure prepared for Data Center
                !=
Fully configured multi-node Bamboo cluster
```

Complete the required application configuration in the Bamboo Setup Wizard according to the intended topology.

## Idempotency

The installation task checks for an existing Bamboo installation before extracting the archive.

On subsequent runs:

- archive extraction should be skipped when the installation and start script already exist;
- package installation remains idempotent;
- user/group and directory tasks converge on the configured state;
- templates only change when their rendered content changes;
- JVM `lineinfile` tasks only change when values differ;
- systemd remains enabled/started;
- validation runs again.

Before declaring the role fully idempotent for a production baseline, run the complete playbook twice and confirm the second execution has no unexpected changes.

Example:

```bash
ansible-playbook   -i inventory/hosts.yml   playbooks/install_bamboo.yml
```

Run the same command again and review:

```text
changed=
failed=
```

## Uninstall Behavior

The uninstall workflow is deliberately conservative.

### Default

With:

```yaml
bamboo_remove_install_dir: true
bamboo_purge_data: false
bamboo_remove_home: false
bamboo_remove_user: false
bamboo_remove_group: false
```

the role:

1. gathers service facts;
2. detects `bamboo.service`;
3. stops Bamboo when present;
4. disables Bamboo;
5. removes `/etc/systemd/system/bamboo.service`;
6. reloads systemd;
7. resets failed systemd state when required;
8. removes `/app/bamboo-12.1.10`;
9. preserves `/app/bamboo-data`;
10. preserves the Bamboo OS user;
11. preserves the Bamboo OS group;
12. preserves PostgreSQL;
13. validates installation-directory removal.

### Full filesystem/account purge

Data/account removal requires the master switch:

```yaml
bamboo_purge_data: true
```

plus the corresponding removal switches.

Example:

```bash
ansible-playbook   -i inventory/hosts.yml   playbooks/uninstall_bamboo.yml   -e bamboo_purge_data=true   -e bamboo_remove_home=true   -e bamboo_remove_user=true   -e bamboo_remove_group=true
```

This can remove:

```text
/app/bamboo-data
bamboo OS user
bamboo OS group
```

### PostgreSQL safety

The uninstall role explicitly preserves:

```text
Database : NOT MODIFIED
DB User  : NOT MODIFIED
```

PostgreSQL lifecycle belongs to the separate PostgreSQL automation.

## Validation

`validate.yml` checks the following.

### Installation directory

Expected:

```text
/app/bamboo-12.1.10
```

### Bamboo Home

Expected:

```text
/app/bamboo-data
```

### systemd service

The role verifies:

```text
bamboo.service exists
systemctl is-active bamboo  = active
systemctl is-enabled bamboo = enabled
```

### HTTP

The role waits up to 300 seconds for:

```text
127.0.0.1:8085
```

and then performs an HTTP GET against:

```text
http://127.0.0.1:8085/
```

The current validation expects HTTP status:

```text
200
```

and retries up to 30 times with a 10-second delay.

### JVM

The role confirms `setenv.sh` exists and validates the configured minimum and maximum JVM memory.

### Expected summary

A successful run ends with a summary similar to:

```text
========================================
Bamboo validation completed
Version           : 12.1.10
Install Directory : /app/bamboo-12.1.10
Home Directory    : /app/bamboo-data
HTTP Port         : 8085
Database          : bamboo
PostgreSQL        : 127.0.0.1:15432
Service           : bamboo
Service State     : active
Service Enabled   : enabled
JVM Xms           : 2g
JVM Xmx           : 4g
Data Center Mode  : True
========================================
Next: Complete setup at http://<server-ip>:8085
Complete the Bamboo Setup Wizard and apply the Data Center license.
```

## Troubleshooting

### Archive not found

Check:

```bash
ls -lh roles/bamboo/files/
```

Expected:

```text
atlassian-bamboo-12.1.10.tar.gz
```

Verify:

```bash
file roles/bamboo/files/atlassian-bamboo-12.1.10.tar.gz
```

and:

```bash
tar -tzf   roles/bamboo/files/atlassian-bamboo-12.1.10.tar.gz   >/dev/null
```

### Verify archive checksum

```bash
sha256sum   roles/bamboo/files/atlassian-bamboo-12.1.10.tar.gz
```

Current supplied checksum:

```text
598937ddac7a781a42a6caf75e564f7cd1a489686e9e4a97415ac2cd4109dda4
```

### `/app` does not exist

The precheck intentionally requires the base directory to exist before prerequisites run.

Check:

```bash
ls -ld /app
df -h /app
```

Create or mount `/app` according to your infrastructure standard before running the playbook.

### Insufficient disk space

Check:

```bash
df -h /app
```

The role requires at least 10 GB available.

### Java validation failure

Check:

```bash
java -version
readlink -f /usr/bin/java
```

The role requires Java 21.

Determine Java home manually with:

```bash
dirname "$(dirname "$(readlink -f /usr/bin/java)")"
```

### PostgreSQL TCP failure

Check:

```bash
nc -vz 127.0.0.1 15432
```

or:

```bash
ss -lntp | grep ':15432'
```

### PostgreSQL authentication failure

Test:

```bash
PGPASSWORD='bamboo' psql   -h 127.0.0.1   -p 15432   -U bamboo   -d bamboo   -c 'select current_database(), current_user;'
```

### Database encoding failure

Check:

```bash
PGPASSWORD='bamboo' psql   -h 127.0.0.1   -p 15432   -U bamboo   -d bamboo   -c "SELECT datname, pg_encoding_to_char(encoding) FROM pg_database WHERE datname='bamboo';"
```

Expected encoding:

```text
UTF8
```

### Database owner failure

Check:

```bash
PGPASSWORD='bamboo' psql   -h 127.0.0.1   -p 15432   -U bamboo   -d bamboo   -c "SELECT datname, pg_catalog.pg_get_userbyid(datdba) AS owner FROM pg_database WHERE datname='bamboo';"
```

Expected owner:

```text
bamboo
```

### Bamboo service does not start

Check:

```bash
systemctl status bamboo --no-pager -l
```

Then:

```bash
journalctl -u bamboo -n 200 --no-pager
```

Check processes:

```bash
ps -ef | grep '[b]amboo'
ps -ef | grep '[j]ava'
```

### Bamboo HTTP port is not listening

Check:

```bash
ss -lntp | grep ':8085'
```

Then inspect service and Bamboo logs.

### HTTP validation fails during first startup

Bamboo initialization can take time.

Check:

```bash
systemctl status bamboo --no-pager -l
ss -lntp | grep ':8085'
curl -I http://127.0.0.1:8085/
```

If the Java process is healthy but Bamboo is still initializing, inspect the application logs before changing timeout values.

### Find Bamboo logs

Start with:

```bash
find /app/bamboo-data   -maxdepth 3   -type f   | sort
```

Then inspect recent log files, for example:

```bash
find /app/bamboo-data   -type f   -name '*.log'   -print
```

### Verify Bamboo Home configuration

```bash
cat   /app/bamboo-12.1.10/atlassian-bamboo/WEB-INF/classes/bamboo-init.properties
```

Expected:

```properties
bamboo.home=/app/bamboo-data
```

### Verify JVM configuration

```bash
grep -n -E   'JVM_MINIMUM_MEMORY|JVM_MAXIMUM_MEMORY'   /app/bamboo-12.1.10/bin/setenv.sh
```

Expected values:

```text
2g
4g
```

### Verbose Ansible troubleshooting

```bash
ansible-playbook   -i inventory/hosts.yml   playbooks/install_bamboo.yml   -vvv
```

## Security Recommendations

- Move `postgres_password` to Ansible Vault.
- Do not commit production passwords to Git.
- Restrict access to inventory files containing secrets.
- Verify the Bamboo archive checksum before deployment.
- Consider adding the known SHA256 as an explicit role variable and enforcing it in `precheck.yml`.
- Back up `/app/bamboo-data` before destructive maintenance.
- Back up PostgreSQL before upgrades or major configuration changes.
- Review `bamboo_purge_data` and all `bamboo_remove_*` flags before uninstall.
- Keep database deletion outside the Bamboo role unless implemented as a separately controlled destructive workflow.
- Restrict Bamboo HTTP access with the appropriate firewall/reverse-proxy design.
- Use TLS at the reverse proxy/load balancer or Bamboo tier according to the deployment architecture.
- Test Bamboo upgrades and Data Center topology changes outside production first.

## Operational Commands

### Service

```bash
systemctl status bamboo --no-pager -l
systemctl start bamboo
systemctl stop bamboo
systemctl restart bamboo
systemctl is-active bamboo
systemctl is-enabled bamboo
```

### Journal

```bash
journalctl -u bamboo -n 100 --no-pager
```

Follow:

```bash
journalctl -u bamboo -f
```

### Process

```bash
ps -ef | grep '[b]amboo'
ps -ef | grep '[j]ava'
```

### Port

```bash
ss -lntp | grep ':8085'
```

### HTTP

```bash
curl -sS -I   --max-time 10   http://127.0.0.1:8085/
```

### Installation ownership

```bash
stat -c '%U:%G %a %n'   /app/bamboo-12.1.10
```

### Bamboo Home ownership

```bash
stat -c '%U:%G %a %n'   /app/bamboo-data
```

### Database connections

```bash
sudo -u postgres psql   -p 15432   -d postgres   -c "SELECT datname, usename, client_addr, state FROM pg_stat_activity WHERE datname='bamboo';"
```

### Database details

```bash
sudo -u postgres psql   -p 15432   -d postgres   -c "\l bamboo"
```

### Database user

```bash
sudo -u postgres psql   -p 15432   -d postgres   -c "SELECT rolname FROM pg_roles WHERE rolname='bamboo';"
```

## Deployment Summary

For the current repository configuration:

```text
Product          : Atlassian Bamboo Data Center
Version          : 12.1.10
Operating System : RHEL 9
Architecture     : x86_64
Java             : OpenJDK 21
Install Method   : tar.gz archive
Install Dir      : /app/bamboo-12.1.10
Bamboo Home      : /app/bamboo-data
HTTP Port        : 8085
PostgreSQL       : 17
Database Host    : 127.0.0.1
Database Port    : 15432
Database         : bamboo
Database User    : bamboo
JVM Xms          : 2g
JVM Xmx          : 4g
Service          : bamboo.service
Data Center      : enabled/prepared
```

The intended lifecycle is:

```text
Prepare RHEL 9 host
        |
        v
Provide Bamboo 12.1.10 archive
        |
        v
Provision PostgreSQL database/user separately
        |
        v
Run Ansible prechecks
        |
        v
Install prerequisites
        |
        v
Extract Bamboo
        |
        v
Validate PostgreSQL
        |
        v
Configure Bamboo Home + JVM
        |
        v
Prepare Data Center mode
        |
        v
Install/start systemd service
        |
        v
Validate HTTP/service/JVM
        |
        v
Complete Bamboo Setup Wizard
        |
        v
Apply Bamboo Data Center license
```

This keeps operating-system provisioning, Bamboo binaries, service management, PostgreSQL validation, validation checks, and uninstall behavior under Ansible control while leaving first-run Bamboo application initialization to the Bamboo Setup Wizard.

# Oracle 19c Data Guard Installation with Ansible

---

## Table of Contents

1. [Overview](#1-overview)
2. [Ansible Project Structure](#2-ansible-project-structure)
3. [Prerequisites](#3-prerequisites)
4. [ISO Repository](#4-iso-repository)
5. [Phase 1: OS Preparation](#5-phase-1-os-preparation)
6. [Phase 2: Grid Infrastructure](#6-phase-2-grid-infrastructure)
7. [Phase 3: DB Software](#7-phase-3-db-software)
8. [Phase 4: Primary Database](#8-phase-4-primary-database)
9. [Phase 5: Standby Database](#9-phase-5-standby-database)
10. [Phase 6: Data Guard Broker](#10-phase-6-data-guard-broker)
11. [Phase 7: rlwrap (Optional)](#11-phase-7-rlwrap-optional)
12. [Completed Tasks](#12-completed-tasks)
13. [Ansible Quick Reference](#13-ansible-quick-reference)

---

## 1. Overview

| Component | Details |
| --- | --- |
| Primary Server | dghost-001 (192.168.178.21) |
| Standby Server | dghost-002 (192.168.178.22) |
| Ansible Controller | anshost (existing AWX Server) |
| Operating System | Oracle Linux 8.10 |
| Oracle Version | 19c Enterprise Edition (19.27.0.0.0) |
| Storage | ASM: sdb (500G) → +DATA, sdc (100G) → +FRA |
| Grid Infra | Oracle 19c GI, Standalone (Oracle Restart) |
| Data Guard Type | Physical Standby |
| RAM | 16 GB per server |
| Oracle User | oracle (uid 54321) — owns GI + DB |
| DG Broker Config | DG_CDBTEST |
| Primary DB | CDBTEST_001 (SID: CDBTEST, PDB: PDBTEST) |
| Standby DB | CDBTEST_002 (SID: CDBTEST) |

---

## 2. Ansible Project Structure

All Ansible files located under `/home/oracle/ansible` on anshost.

```
/home/oracle/ansible/
├── ansible.cfg
├── inventory/
│   └── dataguard-001-002/           # Inventory directory (use as template)
│       ├── hosts                    # Hosts + variables
│       └── group_vars/
│           └── oracle_servers/
│               └── vault.yml        # Encrypted passwords (ansible-vault)
├── playbooks/
│   ├── dataguard/
│   │   ├── 01_os_preparation.yml
│   │   ├── 02_grid_install.yml
│   │   ├── 02_grid_deinstall.yml
│   │   ├── 03_db_install.yml
│   │   ├── 04_create_primary.yml
│   │   ├── 05_create_standby.yml
│   │   ├── 06_dg_broker.yml
│   │   └── 07_install_rlwrap.yml    # Optional: rlwrap + alias activation
│   └── patching/
│       ├── patch_dg_step1_prepare.yml
│       ├── patch_dg_step2_standby.yml
│       ├── patch_dg_step3_switchover.yml
│       ├── patch_dg_step4_primary.yml
│       ├── patch_dg_step5_switchback.yml
│       ├── patch_dg_step6_datapatch.yml
│       ├── patch_dg_step7_validate.yml
│       └── patch_dg_rollback.yml
├── files/
│   └── 19c/
│       ├── LINUX.X64_193000_grid_home.zip
│       ├── LINUX.X64_193000_db_home.zip
│       └── patches/
│           ├── p37641958_190000_Linux-x86-64.zip
│           ├── p37642901_190000_Linux-x86-64.zip
│           ├── p37499406_190000_Linux-x86-64.zip
│           └── p6880880_190000_Linux-x86-64.zip
├── templates/
└── roles/
```

> [!tip] Scalable
> For additional DG pairs: create a new inventory directory under `inventory/` with its own `hosts` and `vault.yml`.
> The playbooks remain identical.

### 2.1 ansible.cfg

```ini
[defaults]
host_key_checking = False
command_warnings = False
vault_password_file = ~/.vault_pass

[privilege_escalation]
become = True
become_method = sudo
```

`vault_password_file` — Ansible reads the vault password automatically from `~/.vault_pass`. This eliminates the need for `--ask-vault-pass` on every playbook run.

### 2.2 Inventory: `inventory/dataguard-001-002/hosts`

```ini
[primary]
dghost-001 ansible_host=192.168.178.21

[standby]
dghost-002 ansible_host=192.168.178.22

[oracle_servers:children]
primary
standby

[oracle_servers:vars]
ansible_user=sef
ansible_python_interpreter=/usr/bin/python3.6

# Oracle User
oracle_user=oracle
oracle_uid=54321
iso_mount=/mnt/iso

# Directory Structure
oracle_base=/u01/app/oracle
oracle_inventory=/u01/app/oraInventory
oracle_home=/u01/app/oracle/product/19.0.0/dbhome_1
grid_home=/u01/app/19.0.0/grid

# Database
oracle_sid=CDBTEST
db_unique_name_pri=CDBTEST_001
db_unique_name_stby=CDBTEST_002
pdb_name=PDBTEST

# Patching
patch_target_version=19.27
patch_dir=/home/oracle/ansible/files/19c/patches
patch_stage=/tmp/patches
gi_patch_zip=p37641958_190000_Linux-x86-64.zip
gi_patch_number=37641958
db_patch_zip=p37642901_190000_Linux-x86-64.zip
db_patch_number=37642901
ojvm_patch_zip=p37499406_190000_Linux-x86-64.zip
ojvm_patch_number=37499406
opatch_zip=p6880880_190000_Linux-x86-64.zip
```

### 2.3 Vault: `inventory/dataguard-001-002/group_vars/oracle_servers/vault.yml`

Passwords are **never** stored in `hosts`. They live in an ansible-vault encrypted file. Ansible automatically loads `group_vars/oracle_servers/` for all hosts in the `oracle_servers` group — no `vars_files` entry needed in any playbook.

Create the file:

```bash
mkdir -p inventory/dataguard-001-002/group_vars/oracle_servers
ansible-vault create inventory/dataguard-001-002/group_vars/oracle_servers/vault.yml
```

Contents (unencrypted):

```yaml
oracle_password: <oracle_os_password>
asm_sys_password: <asm_sys_password>
sys_password: <db_sys_password>
system_password: <db_system_password>
```

Edit the file later:

```bash
ansible-vault edit inventory/dataguard-001-002/group_vars/oracle_servers/vault.yml
```

Store the vault password for automatic loading:

```bash
echo 'your_vault_password' > ~/.vault_pass
chmod 600 ~/.vault_pass
```

For a second DG pair, simply create a new directory with its own independent vault:

```
inventory/
├── dataguard-001-002/
│   ├── hosts
│   └── group_vars/oracle_servers/vault.yml
└── dataguard-003-004/
    ├── hosts
    └── group_vars/oracle_servers/vault.yml
```

---

## 3. Prerequisites

### 3.1 Set Up SSH Access

```bash
ssh-keygen -t rsa -b 4096
ssh-copy-id sef@192.168.178.21
ssh-copy-id sef@192.168.178.22

cd /home/oracle/ansible
ansible -i inventory/dataguard-001-002/ all -m ping -K
```

---

## 4. ISO Repository

Both servers have no internet access. OL 8.10 ISO mounted via vSphere as CD, mounted via fstab:

```
/dev/sr0 /mnt/iso iso9660 ro,nofail 0 0
```

---

## 5. Phase 1: OS Preparation

### Playbook: `01_os_preparation.yml`

| Phase | Description |
| --- | --- |
| 1 | Local Yum repo (BaseOS + AppStream) |
| 2 | oracle-database-preinstall-21c + packages |
| 3 | Oracle groups + user |
| 4 | Directory structure |
| 5 | Kernel parameters (sysctl) |
| 6 | Resource limits |
| 7 | Bash profile |
| 8 | ASM disk partitioning + udev |
| 9 | /etc/hosts |
| 10 | Firewall + SELinux |
| 11 | SSH key for oracle |
| 12 | Validation |

```bash
ansible-playbook -i inventory/dataguard-001-002/ playbooks/dataguard/01_os_preparation.yml -K
```

### Configuration Details

**Directories:**

```
/u01/app/oracle/                         ORACLE_BASE
/u01/app/oracle/product/19.0.0/dbhome_1  ORACLE_HOME
/u01/app/19.0.0/grid/                    GRID_HOME
/u01/app/oraInventory/                   Oracle Inventory
```

**ASM Disks:**

| Device | Partition | Diskgroup | Size |
| --- | --- | --- | --- |
| /dev/sdb | /dev/sdb1 | +DATA | 500G |
| /dev/sdc | /dev/sdc1 | +FRA | 100G |

**Kernel Parameters** (`/etc/sysctl.d/99-oracle.conf`): fs.file-max=6815744, kernel.shmmax=4398046511104, etc.

**Resource Limits** (`/etc/security/limits.d/99-oracle.conf`): nofile 65536, nproc 16384, memlock unlimited.

**Bash Profile** (`/home/oracle/.bash_profile`) — aliases after rlwrap installation:

```bash
alias sqlp='rlwrap sqlplus / as sysdba'
alias rman='rlwrap rman'
alias asmcmd='rlwrap asmcmd'
```

> [!note] rlwrap
> Installed via EPEL (`07_install_rlwrap.yml`). The aliases are initially commented out in the bash profile and activated automatically by the playbook.

---

## 6. Phase 2: Grid Infrastructure

### Playbook: `02_grid_install.yml`

| Phase | Description |
| --- | --- |
| 1-2 | Copy + extract ZIP |
| 3-4 | Response file + silent install |
| 5 | Root scripts |
| 6 | ASM + DATA/FRA diskgroups (ASMCA) |
| 7 | Listener (NETCA) |
| 8-9 | Validation + cleanup |

```bash
ansible-playbook -i inventory/dataguard-001-002/ playbooks/dataguard/02_grid_install.yml -K
```

Deinstallation: `02_grid_deinstall.yml` (no vault required)

### ASM Diskgroups

```
MOUNTED  EXTERN  DATA/  511996 MB  (~500G)
MOUNTED  EXTERN  FRA/   102396 MB  (~100G)
```

### Connect to ASM

```bash
sudo su - oracle
export ORACLE_HOME=/u01/app/19.0.0/grid
export ORACLE_SID=+ASM
$ORACLE_HOME/bin/sqlplus / as sysasm
$ORACLE_HOME/bin/asmcmd
```

> [!warning] Lessons Learned GI
> - Extract GI software as **oracle user**
> - `configToolAllCommands` configures listener only, **not ASM** — run ASMCA separately
> - After deinstallation: reset ownership of GRID_HOME to `oracle:oinstall`

---

## 7. Phase 3: DB Software

### Playbook: `03_db_install.yml`

| Phase | Description |
| --- | --- |
| 1-2 | Copy + extract ZIP |
| 3-4 | Response file + silent install (SWONLY) |
| 5-7 | Root script + validation + cleanup |

```bash
ansible-playbook -i inventory/dataguard-001-002/ playbooks/dataguard/03_db_install.yml -K
```

Installed version: SQL*Plus 19.0.0.0.0 (19.3), OPatch 12.2.0.1.17

---

## 8. Phase 4: Primary Database

### Playbook: `04_create_primary.yml`

4 plays: DBCA + DG Prep | TNS on both servers | Copy password file | Validation

```bash
ansible-playbook -i inventory/dataguard-001-002/ playbooks/dataguard/04_create_primary.yml -K
```

| Parameter | Value |
| --- | --- |
| Global Database Name | CDBTEST_001 |
| SID | CDBTEST |
| PDB | PDBTEST |
| Character Set | AL32UTF8 |
| SGA / PGA | 4G / 1G |
| Redo Logs | 100M (3 groups) |
| Standby Redo Logs | 4 x 100M |
| Storage | +DATA / +FRA |

### Data Guard Parameters

| Parameter | Value |
| --- | --- |
| db_unique_name | CDBTEST_001 |
| log_archive_config | DG_CONFIG=(CDBTEST_001,CDBTEST_002) |
| fal_server | CDBTEST_002 |
| standby_file_management | AUTO |
| dg_broker_start | TRUE |

> [!important]
> `log_archive_dest_2` is **not** set manually — the DG Broker takes over.

### TNS

- `tnsnames.ora` in `ORACLE_HOME/network/admin/` on both servers
- `listener.ora` in `GRID_HOME/network/admin/` with static entries (DGMGRL)

---

## 9. Phase 5: Standby Database

### Playbook: `05_create_standby.yml`

5 plays: Prepare NOMOUNT | RMAN Duplicate | MRP + srvctl | Log shipping | Validation

```bash
ansible-playbook -i inventory/dataguard-001-002/ playbooks/dataguard/05_create_standby.yml -K
```

| Property | Value |
| --- | --- |
| Data Guard Type | Physical Standby |
| Apply Method | Redo Apply (block-for-block) |
| Standby Open Mode | MOUNTED |
| Apply Process | MRP0 |
| RMAN Method | DUPLICATE FROM ACTIVE DATABASE (directly via network) |

> [!warning] Recovery Scenarios
> - **Restore + full recovery**: Standby adjusts automatically
> - **Point-in-time recovery**: Rebuild standby (run playbook 05 again)

---

## 10. Phase 6: Data Guard Broker

### Playbook: `06_dg_broker.yml`

2 plays: Reset `log_archive_dest_2` | Create broker configuration

```bash
ansible-playbook -i inventory/dataguard-001-002/ playbooks/dataguard/06_dg_broker.yml -K
```

```
Configuration - DG_CDBTEST
  Protection Mode: MaxPerformance
  CDBTEST_001 - Primary database       → SUCCESS
  CDBTEST_002 - Physical standby       → SUCCESS
  Transport Lag: 0 seconds
  Apply Lag:     0 seconds
```

### DG Broker Commands

```bash
dgmgrl sys@CDBTEST_001
SHOW CONFIGURATION;
SHOW DATABASE 'CDBTEST_002';
SWITCHOVER TO 'CDBTEST_002';
VALIDATE DATABASE 'CDBTEST_002';
```

> [!warning] Broker and Manual Parameters
> - Broker **takes over** `log_archive_dest_2` — do **not** set manually
> - Before broker setup: `ALTER SYSTEM RESET log_archive_dest_2`, otherwise ORA-16698

---

## 11. Phase 7: rlwrap (Optional)

### Playbook: `07_install_rlwrap.yml`

Installs rlwrap via Oracle EPEL and activates the aliases in the oracle user's bash profile.

```bash
ansible-playbook -i inventory/dataguard-001-002/ playbooks/dataguard/07_install_rlwrap.yml -K
```

| Phase | Description |
| --- | --- |
| 1 | Install Oracle EPEL release package |
| 2 | Install rlwrap + verify version |
| 3 | Activate aliases in `.bash_profile` (uncomment preconfigured lines) |

After installation the following aliases are available:

```bash
sqlp       # rlwrap sqlplus / as sysdba
rman       # rlwrap rman
asmcmd     # rlwrap asmcmd
```

> [!note]
> This playbook requires internet access to reach the EPEL repository. On offline servers, rlwrap must be provided manually as an RPM.

---

## 12. Completed Tasks

| No | Task | Playbook |
| --- | --- | --- |
| 1 | OS preparation | `01_os_preparation.yml` |
| 2 | Grid Infrastructure + ASM | `02_grid_install.yml` |
| 3 | Oracle Database 19c software | `03_db_install.yml` |
| 4 | Primary database + DG prep | `04_create_primary.yml` |
| 5 | Standby database | `05_create_standby.yml` |
| 6 | Data Guard Broker | `06_dg_broker.yml` |
| 7 | rlwrap + aliases | `07_install_rlwrap.yml` |
| 8 | RU Patching (19.3 → 19.27) | `patching/step1-7` |
| 9 | Encrypt passwords with ansible-vault | `group_vars/oracle_servers/vault.yml` |
| 10 | Test rollback playbook | `patch_dg_rollback.yml` |

---

## 13. Ansible Quick Reference

Run all commands from `/home/oracle/ansible`.

| Command | Description |
| --- | --- |
| `ansible -i inventory/dataguard-001-002/ all -m ping -K` | Test connectivity |
| `ansible -i inventory/dataguard-001-002/ all -m shell -a "uptime" -K` | Run single command |
| `ansible-playbook -i inventory/dataguard-001-002/ <playbook> -K` | Run playbook (vault loaded automatically) |
| `ansible-playbook -i inventory/dataguard-001-002/ <playbook> --check -K` | Dry run |
| `ansible-vault create inventory/dataguard-001-002/group_vars/oracle_servers/vault.yml` | Create vault file |
| `ansible-vault edit inventory/dataguard-001-002/group_vars/oracle_servers/vault.yml` | Edit vault file |
| `ansible-doc <module>` | Module documentation |

> [!note] Notes
> - `-K` is required because user `sef` has a sudo password
> - Vault is loaded automatically via `vault_password_file = ~/.vault_pass` in ansible.cfg
> - `--check` may fail on `shell` tasks
> - For new DG pairs: copy the inventory directory, adjust values, reuse playbooks
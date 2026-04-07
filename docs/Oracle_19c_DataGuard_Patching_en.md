# Oracle 19c Data Guard — RU Patching with Ansible

**Inventory:** `inventory/dataguard-001-002/`
**Target version:** 19.27.0.0.0

---

## Table of Contents

1. [Oracle 19c RU Patching](#1-oracle-19c-ru-patching)
   - [1.1 Overview](#11-overview)
   - [1.2 Patching Order](#12-patching-order)
   - [1.3 Playbooks](#13-playbooks)
   - [1.4 Execution — Data Guard](#14-execution--data-guard-zero-downtime)
   - [1.5 Execution — Standalone](#15-execution--standalone-without-data-guard)
   - [1.6 Rollback](#16-rollback)
   - [1.7 Patching Sequence per Server](#17-patching-sequence-per-server)
   - [1.8 Switchover Safety](#18-switchover-safety)
   - [1.9 Result after Patching](#19-result-after-patching)
   - [1.10 Lessons Learned](#110-lessons-learned)
   - [1.11 For Other Patch Versions](#111-for-other-patch-versions)
2. [Completed Tasks](#2-completed-tasks)
3. [Ansible Quick Reference](#3-ansible-quick-reference)

---

## 1. Oracle 19c RU Patching

### 1.1 Overview

Zero downtime patching from 19.3 to 19.27 (April 2025). The database is **always online**.

| Property | Value |
| --- | --- |
| Target version | 19.27.0.0.250415 |
| GI RU Patch | 37641958 |
| DB RU Patch | 37642901 |
| OJVM RU Patch | 37499406 |
| OPatch | 6880880 (12.2.0.1.49) |
| Method | Zero downtime via switchover |
| Total duration | approx. 1.5 - 2 hours |

### 1.2 Patching Order

```
STEP 1: Update OPatch                              (both servers)
STEP 2: Patch Standby (dghost-002)                 Primary (dghost-001) ONLINE
STEP 3: SWITCHOVER → dghost-002 becomes Primary    DB ONLINE
STEP 4: Patch old Primary (dghost-001)             New Primary (dghost-002) ONLINE
STEP 5: SWITCHOVER back → dghost-001 Primary       DB ONLINE
STEP 6: Datapatch on Primary                       DB ONLINE
STEP 7: Validation + cleanup
```

> [!important] Standby first!
> Oracle supports redo apply from a lower-version primary to a higher-version standby, but not vice versa.

### 1.3 Playbooks

```
playbooks/patching/
├── patch_dg_step1_prepare.yml      # Update OPatch (both servers)
├── patch_dg_step2_standby.yml      # Patch standby (GI + DB + OJVM)
├── patch_dg_step3_switchover.yml   # Switchover → standby becomes primary
├── patch_dg_step4_primary.yml      # Patch old primary (also works standalone)
├── patch_dg_step5_switchback.yml   # Switchback → restore original roles
├── patch_dg_step6_datapatch.yml    # Run datapatch
├── patch_dg_step7_validate.yml     # Validation + cleanup
└── patch_dg_rollback.yml           # Full rollback (zero downtime)
```

### 1.4 Execution — Data Guard (Zero Downtime)

```bash
cd /home/oracle/ansible
ansible-playbook -i inventory/dataguard-001-002/ playbooks/patching/patch_dg_step1_prepare.yml -K
ansible-playbook -i inventory/dataguard-001-002/ playbooks/patching/patch_dg_step2_standby.yml -K
ansible-playbook -i inventory/dataguard-001-002/ playbooks/patching/patch_dg_step3_switchover.yml -K
ansible-playbook -i inventory/dataguard-001-002/ playbooks/patching/patch_dg_step4_primary.yml -K
ansible-playbook -i inventory/dataguard-001-002/ playbooks/patching/patch_dg_step5_switchback.yml -K
ansible-playbook -i inventory/dataguard-001-002/ playbooks/patching/patch_dg_step6_datapatch.yml -K
ansible-playbook -i inventory/dataguard-001-002/ playbooks/patching/patch_dg_step7_validate.yml -K
```

### 1.5 Execution — Standalone (without Data Guard)

Steps 2, 3, and 5 are skipped. Step 4 automatically detects whether a standby group exists:

```bash
ansible-playbook -i inventory/standalone-server/ playbooks/patching/patch_dg_step1_prepare.yml -K
ansible-playbook -i inventory/standalone-server/ playbooks/patching/patch_dg_step4_primary.yml -K
ansible-playbook -i inventory/standalone-server/ playbooks/patching/patch_dg_step6_datapatch.yml -K
ansible-playbook -i inventory/standalone-server/ playbooks/patching/patch_dg_step7_validate.yml -K
```

### 1.6 Rollback

```bash
ansible-playbook -i inventory/dataguard-001-002/ playbooks/patching/patch_dg_rollback.yml -K
```

The rollback playbook dynamically detects which host is currently primary and which is standby — independent of the inventory roles. This means it works correctly even after a manual switchover. Rollback order per server: OJVM → DB → GI (reverse of apply order).

### 1.7 Patching Sequence per Server

Each patch ZIP is copied, extracted, applied, then immediately deleted to conserve disk space. Minimum ~15 GB free on `/` required for `opatchauto`.

```
1. GI:   Copy ZIP → extract → delete ZIP → opatchauto apply → delete extracted
2. DB:   Copy ZIP → extract → delete ZIP → opatch apply    → delete extracted
3. OJVM: Copy ZIP → extract → delete ZIP → opatch apply    → delete extracted
```

| Patch | Tool | User | Target Home |
| --- | --- | --- | --- |
| GI RU | `opatchauto` | root | GRID_HOME |
| DB RU | `opatch` | oracle | ORACLE_HOME |
| OJVM RU | `opatch` | oracle | ORACLE_HOME |

### 1.8 Switchover Safety

Before every switchover the playbooks verify:

| Check | Description | On failure |
| --- | --- | --- |
| DG Config SUCCESS | Broker has no errors | Playbook stops |
| MRP APPLY-ON | Redo apply is running | Set automatically |
| Apply Lag = 0 seconds | Standby is synchronized | Waits up to 10 min |
| switchover_status | Standby is ready | Playbook stops |

If a switchover fails, an automatic retry with MRP fix is attempted. If the retry also fails, the playbook stops for manual intervention.

### 1.9 Result after Patching

| Check | dghost-001 (Primary) | dghost-002 (Standby) |
| --- | --- | --- |
| Oracle Version | 19.27.0.0.0 | 19.27.0.0.0 |
| GI RU | 37641958 | 37641958 |
| DB RU | 37642901 | 37642901 |
| OJVM | 37499406 | 37499406 |
| Datapatch | SUCCESS (CDB + PDB) | via Redo |
| DG Broker | SUCCESS | |
| Lag | -- | 0 seconds |

### 1.10 Lessons Learned

> [!warning] Critical Points
> - Before `opatchauto`: kill all Oracle user processes (`pkill -9 -u oracle`), otherwise "active files" error
> - `opatchauto` stops/starts CRS/ASM/Listener automatically — do not do this manually
> - `opatchauto` may start the DB automatically after GI patching → safety check before `opatch`
> - Before switchover: MRP **must be running** and apply lag **must be 0**, otherwise ORA-16516
> - Broker `SUCCESS` does not mean "Apply Lag = 0" — check both separately
> - `until` condition for apply lag: use `regex_search('Apply Lag:\s+0 seconds')` — dgmgrl output has variable whitespace, a literal string comparison will fail
> - `until` conditions in Ansible: always wrap filter expressions in parentheses: `(var | filter) and ('x' in var)`
> - GI patch `37641958` contains sub-patches not applicable to Standalone Oracle Restart — `opatchauto` reports "not required" and skips them — this is correct, not an error

> [!info] Disk Space and OPatch
> - Install OPatch for GRID_HOME as **root**
> - Copy, apply, and delete patches one at a time when disk space is limited
> - Minimum ~15 GB free space on `/` for `opatchauto`

### 1.11 For Other Patch Versions

Only change the patch variables in the inventory `hosts` file:

```ini
# Example for 19.28 (July 2025):
patch_target_version=19.28
gi_patch_zip=p37957391_190000_Linux-x86-64.zip
gi_patch_number=37957391
db_patch_zip=p37960098_190000_Linux-x86-64.zip
db_patch_number=37960098
ojvm_patch_zip=p37847857_190000_Linux-x86-64.zip
ojvm_patch_number=37847857
```

---

## 2. Completed Tasks

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

## 3. Ansible Quick Reference

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
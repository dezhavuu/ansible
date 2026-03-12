# Oracle 19c Data Guard — RU Patching with Ansible

**Inventory:** `inventory/dataguard-001-002`
**Target version:** 19.27.0.0.0

---

## 1. Oracle 19c RU Patching

### 1.1 Overview

Zero downtime patching from 19.3 to 19.27 (April 2025). DB is **always online**.

| Property | Value |
|----------|-------|
| Target version | 19.27.0.0.250415 |
| GI RU Patch | 37641958 |
| DB RU Patch | 37642901 |
| OJVM RU Patch | 37499406 |
| OPatch | 6880880 (12.2.0.1.49) |
| Method | Zero downtime via switchover |
| Total duration | approx. 1.5 - 2 hours |

### 1.2 Patching Order

```
STEP 1: Update OPatch                     (both servers)
STEP 2: Patch standby (002)               ← Primary (001) ONLINE 
STEP 3: SWITCHOVER → 002 becomes primary  ← DB ONLINE 
STEP 4: Patch old primary (001)            ← New primary (002) ONLINE 
STEP 5: SWITCHOVER back → 001 primary     ← DB ONLINE 
STEP 6: Datapatch on primary              ← DB ONLINE 
STEP 7: Validation + cleanup
```

> [!important] Standby first!
> Oracle supports redo apply from a lower primary to a higher standby, but not vice versa.

### 1.3 Playbooks

```
playbooks/patching/
├── patch_dg_step1_prepare.yml      # Update OPatch
├── patch_dg_step2_standby.yml      # Patch standby (GI + DB + OJVM)
├── patch_dg_step3_switchover.yml   # Switchover → standby becomes primary
├── patch_dg_step4_primary.yml      # Patch old primary (or standalone)
├── patch_dg_step5_switchback.yml   # Switchback → restore original roles
├── patch_dg_step6_datapatch.yml    # Run datapatch
├── patch_dg_step7_validate.yml     # Validation + cleanup
└── patch_dg_rollback.yml           # Full rollback (zero downtime)
```

### 1.4 Execution — Data Guard (Zero Downtime)

```bash
cd /home/oracle/ansible
ansible-playbook -i inventory/dataguard-001-002 playbooks/patching/patch_dg_step1_prepare.yml -K
ansible-playbook -i inventory/dataguard-001-002 playbooks/patching/patch_dg_step2_standby.yml -K
ansible-playbook -i inventory/dataguard-001-002 playbooks/patching/patch_dg_step3_switchover.yml -K
ansible-playbook -i inventory/dataguard-001-002 playbooks/patching/patch_dg_step4_primary.yml -K
ansible-playbook -i inventory/dataguard-001-002 playbooks/patching/patch_dg_step5_switchback.yml -K
ansible-playbook -i inventory/dataguard-001-002 playbooks/patching/patch_dg_step6_datapatch.yml -K
ansible-playbook -i inventory/dataguard-001-002 playbooks/patching/patch_dg_step7_validate.yml -K
```

### 1.5 Execution — Standalone (without Data Guard)

Steps 2, 3, 5 are skipped. Step 4 automatically detects whether DG exists:

```bash
ansible-playbook -i inventory/standalone-server playbooks/patching/patch_dg_step1_prepare.yml -K
ansible-playbook -i inventory/standalone-server playbooks/patching/patch_dg_step4_primary.yml -K
ansible-playbook -i inventory/standalone-server playbooks/patching/patch_dg_step6_datapatch.yml -K
ansible-playbook -i inventory/standalone-server playbooks/patching/patch_dg_step7_validate.yml -K
```

### 1.6 Rollback

```bash
ansible-playbook -i inventory/dataguard-001-002 playbooks/patching/patch_dg_rollback.yml -K
```

Rollback order per server: OJVM → DB → GI (reverse of apply order).

### 1.7 Patching Sequence per Server

```
1. GI:   Copy ZIP → extract → delete ZIP → opatchauto apply → delete extracted
2. DB:   Copy ZIP → extract → delete ZIP → opatch apply → delete extracted
3. OJVM: Copy ZIP → extract → delete ZIP → opatch apply → delete extracted
```

| Patch | Tool | User | Target Home |
|-------|------|------|-------------|
| GI RU | `opatchauto` | root | GRID_HOME |
| DB RU | `opatch` | oracle | ORACLE_HOME |
| OJVM RU | `opatch` | oracle | ORACLE_HOME |

### 1.8 Switchover Safety

Before each switchover the playbooks verify:

| Check | Description | On failure |
|-------|-------------|------------|
| DG Config SUCCESS | Broker has no errors | Playbook stops |
| MRP APPLY-ON | Redo apply is running | Will be set |
| Apply Lag = 0 | Standby is synchronized | Waits up to 10 min |
| switchover_status | Standby is ready | Playbook stops |

On switchover failure: automatic retry with MRP fix. On second failure: playbook stops for manual intervention.

### 1.9 Result after Patching

| Check | dghost-001 (Primary) | dghost-002 (Standby) |
|-------|----------------------|----------------------|
| Oracle Version | 19.27.0.0.0 | 19.27.0.0.0 |
| GI RU | 37642901 | 37642901 |
| DB RU | 37642901 | 37642901 |
| OJVM | 37499406 | 37499406 |
| Datapatch | SUCCESS (CDB + PDB) | via Redo |
| DG Broker | SUCCESS | |
| Lag | — | 0 seconds |

### 1.10 Lessons Learned

> [!warning] Critical Points
> - Before `opatchauto`: kill all Oracle user processes (`pkill -9 -u oracle`), otherwise "active files" error
> - `opatchauto` stops/starts CRS/ASM/Listener automatically — do not do this manually
> - `opatchauto` may start DB automatically after GI patching → safety check before `opatch`
> - Before switchover: MRP **must be running** and apply lag **must be 0**, otherwise ORA-16516
> - Broker `SUCCESS` ≠ "Apply Lag = 0" — check both separately

> [!info] Disk Space and OPatch
> - Install OPatch for GRID_HOME as **root**
> - Copy/apply/delete patches one at a time when disk space is limited
> - Minimum ~15 GB free space on `/` for `opatchauto`

### 1.11 For Other Patch Versions

Only change the patch variables in the inventory:

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
|----|------|----------|
| 1 | OS preparation | `01_os_preparation.yml` |
| 2 | Grid Infrastructure + ASM | `02_grid_install.yml` |
| 3 | Oracle Database 19c software | `03_db_install.yml` |
| 4 | Primary database + DG prep | `04_create_primary.yml` |
| 5 | Standby database | `05_create_standby.yml` |
| 6 | Data Guard Broker | `06_dg_broker.yml` |
| 7 | RU Patching (19.3 → 19.27) | `patching/step1-7` |

## 3. Open Tasks

| No | Task |
|----|------|
| 1 | Encrypt passwords with ansible-vault |
| 2 | Install rlwrap (when internet available) |
| 3 | Test rollback playbook |

---

## 4. Ansible Quick Reference

Run all commands from `/home/oracle/ansible`.

| Command | Description |
|---------|-------------|
| `ansible ... -m ping -K` | Test connectivity |
| `ansible ... -m shell -a "..." -K` | Run single command |
| `ansible-playbook ... <playbook> -K` | Run playbook |
| `ansible-playbook ... --check` | Dry run |
| `ansible-vault create secrets.yml` | Encrypt passwords |
| `ansible-doc <module>` | Module documentation |

> [!note] Notes
> - `-K` is required because user `sef` has a sudo password
> - `--check` may fail on `shell` tasks
> - For new DG pairs: copy inventory, adjust values, reuse playbooks

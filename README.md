# Oracle 19c Data Guard Automation with Ansible

Ansible playbooks for fully automated deployment and zero-downtime patching of an Oracle 19c Data Guard environment (Physical Standby) on Oracle Linux 8.

> **Learning project** — built and tested in a VirtualBox lab environment. Designed to be reusable for other Data Guard setups by simply swapping the inventory file.

---

## What This Does

| Area | Description |
| --- | --- |
| OS Preparation | Kernel parameters, resource limits, ASM disk setup, users/groups, firewall |
| Grid Infrastructure | Oracle 19c GI silent install, ASM diskgroups (+DATA, +FRA), Listener |
| Database Software | Oracle 19c DB silent install (SWONLY) |
| Primary Database | DBCA, Data Guard parameters, TNS, password file |
| Standby Database | RMAN active duplicate over network, MRP apply |
| Data Guard Broker | Broker configuration, switchover validation |
| rlwrap | Optional: rlwrap installation + alias activation for sqlplus/rman/asmcmd |
| RU Patching | Zero-downtime patching via switchover (19.3 → 19.27), rollback included |

---

## Environment

| Component | Details |
| --- | --- |
| Primary Server | dghost-001 (192.168.178.21) |
| Standby Server | dghost-002 (192.168.178.22) |
| Ansible Controller | anshost (AWX) |
| OS | Oracle Linux 8.10 |
| Oracle Version | 19c EE (19.27.0.0.0) |
| Storage | ASM: +DATA (500G), +FRA (100G) |
| Data Guard Type | Physical Standby (MaxPerformance) |
| RAM | 16 GB per server |

---

## Project Structure

```
├── ansible.cfg
├── inventory/
│   └── dataguard-001-002/           # Inventory directory (use as template)
│       ├── hosts                    # Hosts + variables
│       └── group_vars/
│           └── oracle_servers/
│               └── vault.yml        # Encrypted passwords (ansible-vault)
├── playbooks/
│   ├── dataguard/                   # Installation playbooks (run in order)
│   │   ├── 01_os_preparation.yml
│   │   ├── 02_grid_install.yml
│   │   ├── 02_grid_deinstall.yml
│   │   ├── 03_db_install.yml
│   │   ├── 04_create_primary.yml
│   │   ├── 05_create_standby.yml
│   │   ├── 06_dg_broker.yml
│   │   └── 07_install_rlwrap.yml    # Optional: rlwrap + alias activation
│   └── patching/                    # Zero-downtime RU patching
│       ├── patch_dg_step1_prepare.yml
│       ├── patch_dg_step2_standby.yml
│       ├── patch_dg_step3_switchover.yml
│       ├── patch_dg_step4_primary.yml
│       ├── patch_dg_step5_switchback.yml
│       ├── patch_dg_step6_datapatch.yml
│       ├── patch_dg_step7_validate.yml
│       └── patch_dg_rollback.yml
├── docs/
│   ├── Oracle_19c_DataGuard_Installation_en.md
│   ├── Oracle_19c_DataGuard_Installation_de.md
│   ├── Oracle_19c_DataGuard_Installation_ru.md
│   ├── Oracle_19c_DataGuard_Patching_en.md
│   ├── Oracle_19c_DataGuard_Patching_de.md
│   └── Oracle_19c_DataGuard_Patching_ru.md
├── files/
│   └── 19c/
│       └── patches/                 # Place Oracle patch ZIPs here (not in Git)
├── LICENSE
└── README.md
```

---

## Quick Start

### 1. Prerequisites

* Ansible installed on the controller node
* Two servers with Oracle Linux 8 (or compatible RHEL-based OS)
* Oracle 19c Grid and Database ZIP files available
* SSH key-based authentication set up

```bash
ssh-keygen -t rsa -b 4096
ssh-copy-id sef@<primary-server>
ssh-copy-id sef@<standby-server>
```

### 2. Configure the Inventory

Copy and adapt the inventory directory for your environment:

```bash
cp -r inventory/dataguard-001-002 inventory/my-dg-setup
```

Edit the key values in `hosts`:

```ini
[primary]
dghost-001 ansible_host=<PRIMARY_IP>

[standby]
dghost-002 ansible_host=<STANDBY_IP>
```

### 3. Set Up Vault for Passwords

**Never store passwords in plain text.** Ansible automatically loads `group_vars/oracle_servers/vault.yml` for all hosts in the `oracle_servers` group — no `vars_files` needed in any playbook.

```bash
mkdir -p inventory/dataguard-001-002/group_vars/oracle_servers
ansible-vault create inventory/dataguard-001-002/group_vars/oracle_servers/vault.yml
```

Add your passwords:

```yaml
oracle_password: "YourOSPassword"
asm_sys_password: "YourASMPassword"
sys_password: "YourSysPassword"
system_password: "YourSystemPassword"
```

Store the vault password for automatic loading:

```bash
echo 'your_vault_password' > ~/.vault_pass
chmod 600 ~/.vault_pass
```

`vault_password_file = ~/.vault_pass` is already set in `ansible.cfg` — no `--ask-vault-pass` needed.

### 4. Run Installation Playbooks (in order)

```bash
cd /home/oracle/ansible

ansible-playbook -i inventory/dataguard-001-002/ playbooks/dataguard/01_os_preparation.yml -K
ansible-playbook -i inventory/dataguard-001-002/ playbooks/dataguard/02_grid_install.yml -K
ansible-playbook -i inventory/dataguard-001-002/ playbooks/dataguard/03_db_install.yml -K
ansible-playbook -i inventory/dataguard-001-002/ playbooks/dataguard/04_create_primary.yml -K
ansible-playbook -i inventory/dataguard-001-002/ playbooks/dataguard/05_create_standby.yml -K
ansible-playbook -i inventory/dataguard-001-002/ playbooks/dataguard/06_dg_broker.yml -K
```

Optional — install rlwrap and activate aliases for `sqlplus`, `rman`, `asmcmd`:

```bash
ansible-playbook -i inventory/dataguard-001-002/ playbooks/dataguard/07_install_rlwrap.yml -K
```

### 5. Zero-Downtime Patching (Data Guard)

Patches GI, DB RU, and OJVM on each server. The database stays online throughout via switchover.

```bash
ansible-playbook -i inventory/dataguard-001-002/ playbooks/patching/patch_dg_step1_prepare.yml -K
ansible-playbook -i inventory/dataguard-001-002/ playbooks/patching/patch_dg_step2_standby.yml -K
ansible-playbook -i inventory/dataguard-001-002/ playbooks/patching/patch_dg_step3_switchover.yml -K
ansible-playbook -i inventory/dataguard-001-002/ playbooks/patching/patch_dg_step4_primary.yml -K
ansible-playbook -i inventory/dataguard-001-002/ playbooks/patching/patch_dg_step5_switchback.yml -K
ansible-playbook -i inventory/dataguard-001-002/ playbooks/patching/patch_dg_step6_datapatch.yml -K
ansible-playbook -i inventory/dataguard-001-002/ playbooks/patching/patch_dg_step7_validate.yml -K
```

**Standalone (no Data Guard):** Steps 2, 3, and 5 are skipped. Step 4 automatically detects whether a standby group exists.

```bash
ansible-playbook -i inventory/standalone-server/ playbooks/patching/patch_dg_step1_prepare.yml -K
ansible-playbook -i inventory/standalone-server/ playbooks/patching/patch_dg_step4_primary.yml -K
ansible-playbook -i inventory/standalone-server/ playbooks/patching/patch_dg_step6_datapatch.yml -K
ansible-playbook -i inventory/standalone-server/ playbooks/patching/patch_dg_step7_validate.yml -K
```

For rollback:

```bash
ansible-playbook -i inventory/dataguard-001-002/ playbooks/patching/patch_dg_rollback.yml -K
```

---

## Patching Details

### Patch order per server

```
STEP 1: Update OPatch                      (both servers)
STEP 2: Patch Standby (dghost-002)         ← Primary (dghost-001) ONLINE
STEP 3: SWITCHOVER → dghost-002 Primary    ← DB ONLINE
STEP 4: Patch old Primary (dghost-001)     ← New Primary (dghost-002) ONLINE
STEP 5: SWITCHOVER back → dghost-001       ← DB ONLINE
STEP 6: Datapatch on Primary               ← DB ONLINE
STEP 7: Validate + Cleanup
```

> **Standby first:** Oracle supports redo apply from a lower-version primary to a higher-version standby, but not the other way around.

### Patches applied per server (copy → unzip → delete ZIP → apply → delete extracted)

| Patch | Tool | User | Target Home |
| --- | --- | --- | --- |
| GI RU | `opatchauto` | root | GRID_HOME |
| DB RU | `opatch` | oracle | ORACLE_HOME |
| OJVM RU | `opatch` | oracle | ORACLE_HOME |

Each patch ZIP is copied, extracted, applied, then immediately deleted to conserve disk space. Minimum ~15 GB free on `/` required for `opatchauto`.

### Switchover safety checks

Before every switchover, the playbooks verify:

| Check | Action on failure |
| --- | --- |
| DG configuration status = SUCCESS | Playbook stops |
| MRP apply state = APPLY-ON | Set automatically |
| Apply Lag = 0 seconds | Waits up to 10 minutes |
| `switchover_status` = TO PRIMARY | Playbook stops |

If a switchover fails, an automatic retry with MRP fix is attempted. If the retry also fails, the playbook stops for manual intervention.

### Rollback

`patch_dg_rollback.yml` performs a full zero-downtime rollback. It dynamically detects which host is currently primary and which is standby — independent of the inventory roles. Rollback order per server: OJVM → DB → GI (reverse of apply).

---

## Documentation

Full step-by-step documentation is available in the `docs/` folder in English, German, and Russian:

* [Installation Guide (EN)](https://github.com/dezhavuu/oracle-ansible/blob/main/docs/Oracle_19c_DataGuard_Installation_en.md)
* [Patching Guide (EN)](https://github.com/dezhavuu/oracle-ansible/blob/main/docs/Oracle_19c_DataGuard_Patching_en.md)

---

## Reusing for Other DG Pairs

The playbooks are fully inventory-driven. To set up a second Data Guard pair:

1. Copy the inventory: `cp -r inventory/dataguard-001-002 inventory/dataguard-003-004`
2. Adjust hostnames, IPs, SIDs, and vault passwords
3. Run the same playbooks pointing to the new inventory

Each inventory directory has its own independent `vault.yml`.

---

## Security Notes

* Passwords are stored exclusively in `ansible-vault` encrypted `vault.yml` files — never in plain text
* Oracle patch ZIP files are excluded from Git via `.gitignore` (too large; download from [My Oracle Support](https://support.oracle.com))
* This project was developed in a lab environment — review and adapt security settings before using in production

---

## License

[MIT](https://github.com/dezhavuu/oracle-ansible/blob/main/LICENSE)
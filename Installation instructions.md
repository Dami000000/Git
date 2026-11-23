# Patching Runbook

This repository contains Ansible playbooks for patching Oracle Identity & Access Management products and infrastructure components. It also includes playbooks for installing and updating Axway API Gateway (AAG) patches.

The playbooks support the installation of Oracle CPU and CSPU patches for Oracle Identity & Access Management products, as well as the deployment of Axway API Gateway patches. They also provide automation for backup, rollback, prerequisite checks, service startup and shutdown, and post-installation verification to ensure a consistent and repeatable patching process across all supported environments.

---

## Table of Contents

- [Patching Runbook](#patching-runbook)
  - [Table of Contents](#table-of-contents)
  - [Prerequisites](#prerequisites)
  - [Directory Structure](#directory-structure)
  - [Pre-patching Checklist](#pre-patching-checklist)
    - [1. Verify and configure `group_vars/vars.yml`](#1-verify-and-configure-group_varsvarsyml)
      - [Domain paths — set per environment](#domain-paths--set-per-environment)
      - [WebLogic passwords](#weblogic-passwords)
      - [Admin Server URLs and ports](#admin-server-urls-and-ports)
      - [Managed server names](#managed-server-names)
    - [2. Verify inventory for the target environment](#2-verify-inventory-for-the-target-environment)
    - [3. Verify the hosts file for the target environment](#3-verify-the-hosts-file-for-the-target-environment)
    - [4. Place patch archives on the Ansible host](#4-place-patch-archives-on-the-ansible-host)
  - [OIG — Oracle Identity Governance](#oig--oracle-identity-governance)
    - [⚠️ NFS Warning](#️-nfs-warning)
    - [Usage](#usage)
    - [Playbook Steps](#playbook-steps)
    - [Requirements Before Running](#requirements-before-running)
    - [Manual steps](#manual-steps)
  - [OAM — Oracle Access Manager](#oam--oracle-access-manager)
    - [⚠️ NFS Warning](#️-nfs-warning-1)
    - [Usage](#usage-1)
    - [Playbook Steps](#playbook-steps-1)
    - [Requirements Before Running](#requirements-before-running-1)
    - [Manual steps](#manual-steps-1)
  - [OUD — Oracle Unified Directory](#oud--oracle-unified-directory)
    - [Notes](#notes)
    - [Usage](#usage-2)
    - [Playbook Steps](#playbook-steps-2)
  - [OHS — Oracle HTTP Server](#ohs--oracle-http-server)
    - [Notes](#notes-1)
    - [Usage](#usage-3)
    - [Playbook Steps](#playbook-steps-3)
  - [OSB — Oracle Service Bus](#osb--oracle-service-bus)
    - [Notes](#notes-2)
    - [Usage](#usage-4)
    - [Playbook Steps](#playbook-steps-4)
    - [Requirements Before Running](#requirements-before-running-2)
  - [AAG — Axway API Gateway](#aag--axway-api-gateway)
    - [Notes](#notes-3)
    - [Manual steps](#manual-steps-2)
    - [Usage](#usage-5)
    - [Playbook Steps](#playbook-steps-5)
  - [Variables Reference](#variables-reference)
    - [Variables that must be set manually per environment (`group_vars/vars.yml`)](#variables-that-must-be-set-manually-per-environment-group_varsvarsyml)
    - [Variables set automatically per inventory group](#variables-set-automatically-per-inventory-group)
    - [Variables that are static (defined in `group_vars/vars.yml`, change only with product upgrade)](#variables-that-are-static-defined-in-group_varsvarsyml-change-only-with-product-upgrade)
  - [Important Note Regarding OPatch Warnings](#important-note-regarding-opatch-warnings)

---

## Prerequisites

- Ansible >= 2.9
- SSH access to target servers as `oracle` user
- Patch archives placed in `/app/patches/` on the Ansible host
- For OIG and OAM: NFS `/mnt/nfs` mounted on both nodes, so when performing backup on one of the nodes, on the second node managed servers must be down (this limitation is only for backup, but with the flag '--skip-tags backup' this limitation doesn't apply)
- After performing the installation of the patch for OIG and OAM, manual steps from the INM documentation is required
- When there will be warning after patch installations You can ignore it as it would indicate that this patch was already installed or the package is not required for given Fusion Middleware
- Ansible host must have the configuration which enables him to switch on Oracle User on servers where the patches are being installed. In order to achieve this please perform steps from the documentation CCN2-CINM-CCN2ng-SS-CCN2 Platform Testing Tools and Stepping Stone - Installation Procedure, chapter: 5.2.1.5.1.7.1	SSH key pair generation.

---

## Directory Structure

```
.
├── group_vars/
│   └── vars.yml               # Global variables for all environments
├── inventory/
│   ├── fat/
│   │   ├── hosts              # FAT hosts file
│   │   └── group_vars/
│   │       ├── all/vars       # FAT-wide variables (env: fat)
│   │       ├── oig_mh/vars    # ap: mh
│   │       ├── oam_ap1/vars   # ap: ap1
│   │       ├── oud_mh/vars    # ap: mh
│   │       └── ...
│   ├── dev/
│   │   ├── hosts              # DEV hosts file
│   │   └── group_vars/
│   │       ├── all/vars       # DEV-wide variables (env: dev)
│   │       └── ...
│   ├── sat/
│   ├── tring/
│   ├── supp/
│   └── prod/
├── playbooks/                 # All playbooks for AAG and Oracle
│   ├── patch_oig.yml               
│   ├── patch_oam.yml
│   ├── patch_oud.yml
│   ├── patch_ohs.yml
│   ├── patch_osb.yml
│   └── patch_aag.yml
└── roles/
    ├── stop_cots/             # stop AAG and Oracle applications
    ├── start_cots/            # start AAG and Oracle applications
    ├── backup/                # backup AAG and Oracle applications
    ├── jdk/                   # JDK installation for Oracle
    ├── opatch_install/        # Opatch installation for Oracle
    ├── spbat/                 # Bandle patches installation for OIG,OAM,OSB
    └── patch/                 # One-off patches installation for Oracle
```

---

## Pre-patching Checklist

### 1. Verify and configure `group_vars/vars.yml`

> **This is the most critical step.** All product-specific parameters are defined here.  
> Review every section relevant to the product(s) being patched before execution.

#### Domain paths — set per environment

OIG and OAM domain names differ between environments. Set the correct values before running:

```yaml
# OIG domain paths
# FAT/TRING
oig_managed_server_domain: /app/config/mserver/domains/IAMGovernanceDomain 

# DEV/SUPP/SAT/CONF/PROD
oig_managed_server_domain: /app/config/mserver/domains/IAMGovernanceDomain12c 

# FAT/TRING 
oig_admin_server_domain: /mnt/nfs/config/aserver/domains/IAMGovernanceDomain   

# DEV/SUPP/SAT/CONF/PROD
oig_admin_server_domain: /mnt/nfs/config/aserver/domains/IAMGovernanceDomain12c  

# OAM domain paths
#FAT/TRING/DEV/SUPP/SAT/CONF/PROD
oam_managed_server_domain: /app/config/mserver/domains/IAMAccessDomain
oam_admin_server_domain: /mnt/nfs/config/aserver/domains/IAMAccessDomain
```

#### WebLogic passwords

Passwords for OIG, OAM and OSB Admin Servers must be set correctly.  

```yaml
# In group_vars/vars.yml — encrypt with ansible-vault:
oig_wls_password: "CHANGE_ME_USE_VAULT"
oam_wls_password: "CHANGE_ME_USE_VAULT"
osb_wls_password: "CHANGE_ME_USE_VAULT"
```

#### Admin Server URLs and ports

Verify that the Admin Server ports match the target environment:

```yaml
# OIG
oig_admin_server_port: 7101
oig_managed_server_port: 14000
oig_soa_port: 8001

# OAM
oam_admin_server_port: 7001
oam_managed_server_port: 14100
oam_ama_server_port: 14150

# OSB
osb_admin_server_port: 7001
osb_managed_server_port: 7010
```

#### Managed server names

Verify that the managed server names match the WebLogic domain configuration:

```yaml
# OIG managed servers
oig_managed_servers_node1:
  - managed_server_node1: wls_oig1
  - managed_server_node1: wls_soa1

oig_managed_servers_node2:
  - managed_server_node2: wls_oig2
  - managed_server_node2: wls_soa2

# OAM managed servers
oam_managed_servers_node1:
  - managed_server_node1: wls_oam1
  - managed_server_node1: wls_ama1

oam_managed_servers_node2:
  - managed_server_node2: wls_oam2
  - managed_server_node2: wls_ama2
```

---

### 2. Verify inventory for the target environment

Each environment has its own inventory directory under `inventory/`. The `group_vars` subdirectory inside each environment contains per-group variables — primarily the `ap` (access point / location) variable.

**Check that the correct `ap` value is set for the group being patched:**

```
inventory/<env>/group_vars/
├── all/vars              → env: <environment_name>   (e.g. env: fat)
├── oig_mh/vars           → ap: mh
├── oam_ap1/vars          → ap: ap1
├── oam_ap2/vars          → ap: ap2
├── oam_ap3/vars          → ap: ap3
├── oam_ap4/vars          → ap: ap4
├── oam_...
├── oud_mh/vars           → ap: mh
├── oud_ap1/vars          → ap: ap1
├── oud_ap2/vars          → ap: ap2
├── oud_ap3/vars          → ap: ap3
├── oud_...
├── ohs_ap1/vars          → ap: ap1
├── ohs_ap2/vars          → ap: ap2
├── ohs_...
├── osb_ap1/vars          → ap: ap1
├── osb_...
├── aag_ap1/vars          → ap: ap1
├── aag_ap2/vars          → ap: ap2
├── aag_...
```

Verify the `all/vars` file sets the correct environment name:

```yaml
# inventory/{{ env }}/group_vars/all/vars
env: {{ env }}
```

To verify the inventory is parsed correctly before running a playbook:

```bash
# List all hosts in the target group
ansible -i inventory/{{ env }}/hosts oig_mh --list-hosts
```
You can check all the configuration with this commend:
```bash
# Check what variables are resolved for a specific host
ansible -i inventory/{{ env }}/hosts oim1.mh.{{ env }}.ccn2.taxud -m debug -a "var=hostvars[inventory_hostname]"
```

---

### 3. Verify the hosts file for the target environment

The `hosts` file defines all server groups and hostnames for the environment.  
**Before running any playbook, verify that:**

- All expected servers are listed under the correct group
- Hostnames match the actual DNS names of the target servers
- Groups are correctly nested under parent groups (`:children`)

Example structure in `inventory/fat/hosts`:

```ini
[all:vars]
ansible_user=oracle
ansible_connection=ssh

[oig_mh]
oim1.mh.fat.ccn2.taxud
oim2.mh.fat.ccn2.taxud

[oig:children]
oig_mh

[oam_ap1]
oam1.ap1.fat.ccn2.taxud
oam2.ap1.fat.ccn2.taxud

[oam:children]
oam_ap1
oam_ap2
oam_ap3
oam_ap4
...
```

Verify SSH connectivity to all hosts in the target group before running the playbook (FAT example):

```bash
# Test SSH connectivity for OIG MH
ansible -i inventory/fat/hosts oig_mh -m ping

# Test for a specific location for OAM AP1
ansible -i inventory/fat/hosts oam_ap1 -m ping

# Test for all hosts
ansible -i inventory/fat/hosts all -m ping
```

---

### 4. Place patch archives on the Ansible host

```
/app/patches/
├── JAVA/    → jdk-8u491-linux-x64.tar.gz
│            → p28186730_1394218_Generic.zip   (OPatch)
├── OIG/     → p34986147_122140_Linux-x86-64.zip   (SPBAT bundle)
│            → p36946553_122140_Generic.zip
├── OAM/     → p37478538_122140_Linux-x86-64.zip   (SPBAT bundle)
│            → p36946553_122140_Generic.zip
│            → p36739303_12214221222_Generic.zip
├── OUD/     → OUD patch archives
├── OHS/     → OHS patch archives
├── OSB/     → p34980707_122140_Generic.zip    (SPBAT bundle)
│            → additional OSB patch archives
└── AAG/     → APIGateway_7.7.20240228_Core_linux-x86-64_BN01.tar.gz
```

---

## OIG — Oracle Identity Governance

### ⚠️ NFS Warning

The Admin Server and its configuration reside on NFS (`/mnt/nfs/config/aserver`).  
**Before performing an NFS backup on oim1 server, the Node Manager on oim2 server must be stopped**, because Node Manager holds open file handles on the NFS resource and causes the backup to hang.

The correct order of operations is handled automatically by the playbook:
1. Stop OIG managed servers (oim1 + oim2) — via WLST with `force=true`
2. Stop Admin Server (oim1)
3. Stop Node Manager (both nodes)
4. Backup
5. Patching
6. Start Node Manager → Admin Server → SOA → OIG

### Usage

To install the patch execute this command:
```bash
# {{ env }} environment — mh location 
ansible-playbook -i inventory/{{ env }}/hosts playbooks/patch_oig.yml -e "target=oig_mh" -e "serial=2"
```

You can also run other commands with different configuration if necessary:
```bash
# Single server
ansible-playbook -i inventory/{{ env }}/hosts playbooks/patch_oig.yml -e "target=oim1.mh.{{ env }}.ccn2.taxud"

# One node at a time 
ansible-playbook -i inventory/{{ env }}/hosts playbooks/patch_oig.yml -e "target=oig_mh" -e "serial=1"
```

### Playbook Steps

| Step | Tag | Description |
|------|-----|-------------|
| Stop OIG services | `stop`, `cots` | Stops SOA+OIG managed servers (force), Admin Server, Node Manager |
| Backup OIG | `backup` | Backup of Oracle Home and domain configuration |
| Install/Update JDK | `jdk` | JDK upgrade |
| Install/Update OPatch | `opatch` | OPatch tool upgrade |
| Apply SPBAT Bundle Patch | `spbat` | OIG Bundle Patch installation |
| Apply OIG patches | `patch` | Additional OIG patches installation |
| Start OIG services | `start`, `cots` | Starts Node Manager → Admin Server → SOA → OIG |

### Requirements Before Running

- Oracle Database **must be running**
- NFS `/mnt/nfs` mounted and accessible — verify: `df -h | grep mnt`
- `oig_managed_server_domain` and `oig_admin_server_domain` set correctly in `group_vars/vars.yml`

### Manual steps

After performing the installation of the patch for OIG, manual steps from the CCN2 Platform Installation Manual (v30.80)  documentation are required. Perform all the actions from those chapters:
- 4.6.2.1.11.	Clearing cache and tmp files
- 4.6.2.1.12.	Post-patching steps for 34986147: IDM STACK PATCH BUNDLE 12.2.1.4.250115

---

## OAM — Oracle Access Manager

### ⚠️ NFS Warning

Same as OIG — the OAM Admin Server resides on NFS (`/mnt/nfs/config/aserver/domains/IAMAccessDomain`).  
**Before an NFS backup on oam1 server, the Node Manager on oam2 server must be stopped.**  
The playbook handles this automatically in the correct order.

### Usage

To install the patch execute this command on all Access Points separatelly:
```bash
# {{ env }} — {{ ap }} location
ansible-playbook -i inventory/{{ env }}/hosts playbooks/patch_oam.yml -e "target=oam_{{ ap }}" -e "serial=2"
```

You can also run other commands with different configuration if necessary:
```bash
# All locations at once
ansible-playbook -i inventory/{{ env }}/hosts playbooks/patch_oam.yml

# Single server
ansible-playbook -i inventory/{{ env }}/hosts playbooks/patch_oam.yml -e "target=oam1.{{ ap }}.{{ env }}.ccn2.taxud"
```

### Playbook Steps

| Step | Tag | Description |
|------|-----|-------------|
| Stop OAM services | `stop`, `cots` | Stops OAM/AMA managed servers, Admin Server, Node Manager |
| Backup OAM | `backup` | Backup of Oracle Home and domain configuration |
| Install/Update JDK | `jdk` | JDK upgrade |
| Install/Update OPatch | `opatch` | OPatch tool upgrade |
| Apply SPBAT Bundle Patch | `spbat` | OAM Bundle Patch installation |
| Apply OAM patches | `patch` | Additional OAM patches installation |
| Start OAM services | `start`, `cots` | Starts Node Manager → Admin Server → OAM → AMA |

### Requirements Before Running

- Oracle Database **must be running**
- NFS `/mnt/nfs` mounted and accessible
- `oam_managed_server_domain` and `oam_admin_server_domain` set correctly in `group_vars/vars.yml`

### Manual steps

After performing the installation of the patch for OAM, manual step from the CCN2 Platform Installation Manual (v30.80) documentation is required. Perform all the actions from this chapter:
- 4.6.1.2.9.	Branding the OAM Interface (Add the Legal Notice Link on the OAM Login Page)

---

## OUD — Oracle Unified Directory

### Notes

- Patched **1 node at a time** by default (`serial=1`) to maintain LDAP service availability (parameter `-e "serial=2"` can be added for environments where temporary unavailability is acceptable) 
- LDAP OUD instance is installed under this path: `/app/ldap_instance`

### Usage

To install the patch execute this command on MH and all Access Points separatelly:
```bash
# {{ env }} — mh location  
ansible-playbook -i inventory/{{ env }}/hosts playbooks/patch_oud.yml -e "target=oud_mh"

# {{ env }} — {{ ap }} location
ansible-playbook -i inventory/{{ env }}/hosts playbooks/patch_oud.yml -e "target=oud_{{ ap }}"
```

You can also run other commands with different configuration if necessary:
```bash
# {{ env }} — all locations
ansible-playbook -i inventory/{{ env }}/hosts playbooks/patch_oud.yml

# Single server
ansible-playbook -i inventory/{{ env }}/hosts playbooks/patch_oud.yml -e "target=oud1.{{ ap }}.{{ env }}.ccn2.taxud"

# Two nodes in parallel (risk — LDAP service unavailability)
ansible-playbook -i inventory/{{ env }}/hosts playbooks/patch_oud.yml -e "target=oud_{{ ap }}" -e "serial=2"
```

### Playbook Steps

| Step | Tag | Description |
|------|-----|-------------|
| Stop OUD services | `stop`, `cots` | Stops the OUD LDAP instance |
| Backup OUD | `backup` | Backup of Oracle Home and LDAP instance |
| Install/Update JDK | `jdk` | JDK upgrade |
| Install/Update OPatch | `opatch` | OPatch tool upgrade |
| Apply OUD patches | `patch` | OUD patches installation |
| Start OUD services | `start`, `cots` | Starts the OUD LDAP instance |

---

## OHS — Oracle HTTP Server

### Notes

- OHS is the HTTP frontend — patched **1 node at a time** (`serial=1`)
- parameter `-e "serial=2"` can be added for environments where temporary unavailability is acceptable. 

### Usage

To install the patch execute this command on all Access Points separatelly:
```bash
# {{ env }} — {{ ap }} location
ansible-playbook -i inventory/{{ env }}/hosts playbooks/patch_ohs.yml -e "target=ohs_{{ ap }}"
```

You can also run other commands with different configuration if necessary:
```bash
# {{ env }} — all locations
ansible-playbook -i inventory/{{ env }}/hosts playbooks/patch_ohs.yml

# Single server
ansible-playbook -i inventory/fat/hosts playbooks/patch_ohs.yml -e "target=ohs1.{{ ap }}.{{ env }}.ccn2.taxud"
```

### Playbook Steps

| Step | Tag | Description |
|------|-----|-------------|
| Stop OHS services | `stop`, `cots` | Stops Oracle HTTP Server |
| Backup OHS | `backup` | Backup of Oracle Home and configuration |
| Install/Update JDK | `jdk` | JDK upgrade |
| Install/Update OPatch | `opatch` | OPatch tool upgrade |
| Apply OHS patches | `patch` | OHS patches installation |
| Start OHS services | `start`, `cots` | Starts Oracle HTTP Server |

---

## OSB — Oracle Service Bus

### Notes

- OSB Admin Server and Managed Servers reside on local filesystem (no NFS)
- Patched **1 node at a time** by default (`serial=1`)
- Parameter `-e "serial=2"` can be added for environments where temporary unavailability is acceptable. 

### Usage

To install the patch execute this command on all Access Points separatelly:
```bash
# {{ env }} — {{ ap }} location
ansible-playbook -i inventory/{{ env }}/hosts playbooks/patch_osb.yml -e "target=osb_{{ ap }}"
```

You can also run other commands with different configuration if necessary:
```bash
# {{ env }} — all locations
ansible-playbook -i inventory/{{ env }}/hosts playbooks/patch_osb.yml

# Single server
ansible-playbook -i inventory/{{ env }}/hosts playbooks/patch_osb.yml -e "target=osb1.{{ ap }}.{{ env }}.ccn2.taxud"
```

### Playbook Steps

| Step | Tag | Description |
|------|-----|-------------|
| Stop OSB services | `stop`, `cots` | Stops OSB managed servers and Admin Server |
| Backup OSB | `backup` | Backup of Oracle Home and domain configuration |
| Install/Update JDK | `jdk` | JDK upgrade |
| Install/Update OPatch | `opatch` | OPatch tool upgrade |
| Apply SPBAT Bundle Patch | `spbat` | SOA/OSB Bundle Patch installation |
| Apply OSB patches | `patch` | Additional OSB patches installation |
| Start OSB services | `start`, `cots` | Starts Admin Server and OSB managed servers |

### Requirements Before Running

- Oracle Database **must be running**
---

## AAG — Axway API Gateway

### Notes

- Patched **1 node at a time** by default (`serial=1`)
- Parameter `-e "serial=2"` can be added for environments where temporary unavailability is acceptable.
- Patch archive: single `.tar.gz` file
- The script applies two workarounds. One for displaying KPS and the second for parsing big messages. No manual modification is required.

### Manual steps

After performing the installation of the patch for AAG, manual step from the CCN2 Platform Installation Manual (v30.80) documentation is required. Perform all the actions from this chapter:
- In section 4.6.1.3.1.5 – Log4j2 Backup and Patching Axway API Gateway 7.7.20250228, please perform steps 5, 6, 7, and 8.

### Usage

To install the patch execute this command on all Access Points separatelly:
```bash
# {{ env }} — {{ ap }} location
ansible-playbook -i inventory/{{ env }}/hosts playbooks/patch_aag.yml -e "target=aag_{{ ap }}"
```

You can also run other commands with different configuration if necessary:
```bash
# {{ env }} — all locations
ansible-playbook -i inventory/{{ env }}/hosts playbooks/patch_aag.yml

# Single server
ansible-playbook -i inventory/{{ env }}/hosts playbooks/patch_aag.yml -e "target=aag1.{{ ap }}.{{ env }}.ccn2.taxud"
```

### Playbook Steps

| Step | Tag | Description |
|------|-----|-------------|
| Stop AAG services | `stop`, `cots` | Stops Axway API Gateway |
| Backup AAG | `backup` | Backup of AAG installation directory |
| Apply AAG patch | `patch` | AAG new version installation |
| Apply AAG Workaround | `workaround` | Workaround after patch installation |
| Start AAG services | `start`, `cots` | Starts Axway API Gateway |

---



## Variables Reference

### Variables that must be set manually per environment (`group_vars/vars.yml`)

| Variable | Description | Example value |
|----------|-------------|---------------|
| `oig_managed_server_domain` | Path to OIG managed server domain | `/app/config/mserver/domains/IAMGovernanceDomain` |
| `oig_admin_server_domain` | Path to OIG admin server domain (NFS) | `/mnt/nfs/config/aserver/domains/IAMGovernanceDomain` |
| `oam_managed_server_domain` | Path to OAM managed server domain | `/app/config/mserver/domains/IAMAccessDomain` |
| `oam_admin_server_domain` | Path to OAM admin server domain (NFS) | `/mnt/nfs/config/aserver/domains/IAMAccessDomain` |
| `oig_wls_password` | WebLogic password for OIG Admin Server | plain text |
| `oam_wls_password` | WebLogic password for OAM Admin Server | plain text |
| `osb_wls_password` | WebLogic password for OSB Admin Server | plain text |
| `jdk_archive` | JDK archive filename | `jdk-8u491-linux-x64.tar.gz` |
| `opatch_archive` | OPatch archive filename | `p28186730_1394218_Generic.zip` |
| `oig_bundle_archive` | OIG SPBAT bundle archive filename | `p34986147_122140_Linux-x86-64.zip` |
| `oam_bundle_archive` | OAM SPBAT bundle archive filename | `p37478538_122140_Linux-x86-64.zip` |
| `osb_bundle_archive` | OSB SPBAT bundle archive filename | `p34980707_122140_Generic.zip` |
| `aag_patch_archive` | AAG patch archive filename | `APIGateway_7.7.20240228_Core_linux-x86-64_BN01.tar.gz` |

### Variables set automatically per inventory group

| Variable | File | Description |
|----------|------|-------------|
| `ap` | `inventory/<env>/group_vars/<group>/vars` | Location identifier (e.g. `mh`, `ap1`, `ap2`) |
| `env` | `inventory/<env>/group_vars/all/vars` | Environment name (e.g. `fat`, `dev`, `prod`) |

### Variables that are static (defined in `group_vars/vars.yml`, change only with product upgrade)

| Variable | Description |
|----------|-------------|
| `oracle_user` | OS user running Oracle processes (`oracle`) |
| `oracle_home_oig` | Oracle Home path for OIG binaries |
| `oracle_home_oam` | Oracle Home path for OAM binaries |
| `oig_admin_server_port` | OIG Admin Server port (`7101`) |
| `oig_managed_server_port` | OIG managed server port (`14000`) |
| `oig_soa_port` | SOA managed server port (`8001`) |
| `oam_admin_server_port` | OAM Admin Server port (`7001`) |
| `oam_managed_server_port` | OAM managed server port (`14100`) |
| `oig_managed_servers_node1` | List of OIG managed server names on node1 |
| `oig_managed_servers_node2` | List of OIG managed server names on node2 |
| `oam_managed_servers_node1` | List of OAM managed server names on node1 |
| `oam_managed_servers_node2` | List of OAM managed server names on node2 |

---

## Important Note Regarding OPatch Warnings

During the installation of Oracle CPU, CSPU, Bundle Patches, or one-off patches, OPatch may display warning messages related to:

- duplicate patches already installed in the Oracle Home,
- patches superseded by newer patches,
- optional components not present in the Oracle Home,
- patches targeting components that are not installed in the environment.

These warnings are expected and do not indicate a failed installation. They can be safely ignored, and the installation of the remaining patches should continue as planned.
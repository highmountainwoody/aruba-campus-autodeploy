# Aruba CX Campus Stand-up Automation (Ansible + Excel)

This project automates Aruba CX campus switch configuration using an Excel workbook as the source of truth. It is designed for a **non‑EVPN** L2 campus with a flattened VSX core. Engineers only manually set the management IP, management gateway, and admin password on each switch. Everything else is automated.

## What this project does

* Converts a customer Excel workbook into Ansible inventories and variables.
* Renders full device configuration files (render‑only mode).
* Bootstraps AOS‑CX API access over SSH for first‑time deployment.
* Deploys configuration via the AOS‑CX HTTP API.
* Validates post‑deployment state and produces a concise markdown report.
* Stores rendered configs, backups, and reports in timestamped artifacts.

## Supported designs (v1)

* **Campus**: Aruba CX campus **without** EVPN/VXLAN.
* **Core**: Aruba CX VSX pair (model supplied in workbook).
* **Access**: Aruba CX 6200F or 6300M stacks.
* **STP**: MST only.
* **VLAN policy**: all VLANs everywhere (strict, no pruning).
* **Gateway mode (site‑level)**:
  * **CORE** (default): core provides SVIs for VLANs defined in workbook.
  * **FIREWALL**: core only has management SVI; no user SVIs.

## L2 mode vs L3 mode (v2)

The project supports two campus modes. You select the mode in the workbook with a **site‑level flag** (for example, `design_mode = L2` or `design_mode = L3`).

### L2 mode (default)

* L2 trunks from access to core.
* SVIs live on the core VSX pair (active‑gateway).
* VLANs are created everywhere (core and access).
* STP/MST is the control plane between core and access.

### L3 mode (v2)

* Routed L3 links between core and access (OSPF Area 0, p2p).
* Access switches **run OSPF** and advertise connected VLANs.
* SVIs remain on the core VSX pair (active‑gateway); access only has the management SVI.
* Management VLAN remains L2 for out‑of‑band reachability; user VLANs are routed.
* ISLs carry **only** the management VLAN in L3 mode.

### How to use the mode switch

1. Set the site‑level flag in the workbook (e.g., `design_mode = L2` or `design_mode = L3`).
2. Provide the additional L3 link addressing fields required for L3 mode (e.g., /31 point‑to‑point links).
3. Run the normal workflow (`excel_to_yaml.py` → `build_config.yml`/`deploy.yml`).

> If the design mode is **L2**, the playbooks render L2 trunk snippets.  
> If the design mode is **L3**, the playbooks render L3 routed link + OSPF snippets.

### Non‑goals for v1

* EVPN/VXLAN campus
* Full ClearPass 802.1X policy implementation
* VLAN pruning per closet
* L3 routed access

## Prerequisites

### Software

* Python 3.10+
* Ansible 2.14+
* Aruba AOS‑CX collection
* Python packages listed in `requirements.txt`

Install Python dependencies:

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
```

Install Ansible collections:

```bash
ansible-galaxy collection install arubanetworks.aoscx ansible.netcommon
```

### Switch prerequisites (Day‑0 manual steps)

On **each** switch, only these steps are required:

1. Set management IP and gateway.
2. Set the admin password.

The bootstrap playbook (`playbooks/bootstrap.yml`) then enables the API and creates the automation user.

## Excel workbook source of truth

The Excel workbook defines the entire campus. See **`docs/excel-template-outline.md`** for the exact tab and column definitions.

Workflow:

1. Copy `docs/excel-template-outline.md` into a customer workbook (or create a new workbook following the outline).
2. Run validation:

```bash
python tools/validate_workbook.py --workbook <workbook>.xlsx
```

3. Convert to inventory:

```bash
python tools/excel_to_yaml.py --workbook <workbook>.xlsx --output inventories/<site_name>
```

## Inventory structure

The converter builds the following layout:

```
inventories/<site_name>/
  hosts.yml
  group_vars/
    all.yml
    core.yml
    access.yml
  host_vars/
    <device_name>.yml
```

## Playbooks

> All playbooks expect `-i inventories/<site_name>/hosts.yml`.

### 1) Render configs (no device changes)

```bash
ansible-playbook -i inventories/<site_name>/hosts.yml playbooks/build_config.yml
```

Outputs rendered configs to:

```
artifacts/<site>/<timestamp>/rendered/
```

> Render-only builds run locally and do **not** require switch reachability.

### 2) Bootstrap API access (SSH / network_cli)

```bash
ansible-playbook -i inventories/<site_name>/hosts.yml playbooks/bootstrap.yml \
  -e "automation_username=ansible" -e "automation_password=SuperSecret"
```

### 3) Deploy via AOS‑CX API (httpapi)

```bash
ansible-playbook -i inventories/<site_name>/hosts.yml playbooks/deploy.yml \
  -e "automation_username=ansible" -e "automation_password=SuperSecret"
```

### 4) Validate and generate report

```bash
ansible-playbook -i inventories/<site_name>/hosts.yml playbooks/validate.yml
```

Outputs a markdown report to:

```
artifacts/<site>/<timestamp>/reports/validation_report.md
```

### 5) Rollback (restore from backup)

```bash
ansible-playbook -i inventories/<site_name>/hosts.yml playbooks/rollback.yml \
  -e "backup_file=artifacts/<site>/<timestamp>/backups/core-1.cfg"
```

## Validation behavior

The validation playbook checks at minimum:

* VSX status/health (core)
* Access stack health
* LACP uplinks up
* VLAN presence on all switches
* MST configuration consistency
* Gateway mode compliance:
  * CORE: SVIs exist on core when specified
  * FIREWALL: only management SVI exists on core

Suggested manual checks are listed in `docs/validate_show_commands.md`.

## Troubleshooting

* **Workbook errors**: run `tools/validate_workbook.py` to see missing or invalid fields.
* **Authentication failures**: confirm the automation user exists and API is enabled.
* **Template errors**: run `build_config.yml` first; review rendered files in `artifacts/`.
* **HTTP API connectivity**: verify switch management IPs and gateway reachability.

## Secrets handling

Passwords and secrets should **not** be committed. Provide credentials via:

* Extra variables: `-e automation_password=...`
* Environment variables + Ansible Vault if desired

## Repository layout

```
playbooks/
  build_config.yml
  bootstrap.yml
  deploy.yml
  validate.yml
  rollback.yml
roles/
  common_system/
  campus_core/
  campus_access/
  campus_services/
  nac_skeleton/
  validate/
templates/
  core_vsx.j2
  access_stack.j2
  snippets/
    vlans.j2
    svis.j2
    mst.j2
    uplinks_l2_trunks.j2
    aaa_clearpass_stub.j2
tools/
  excel_to_yaml.py
  validate_workbook.py
docs/
  excel-template-outline.md
  validate_show_commands.md
inventories/
  <site_name>/...
artifacts/
  <site>/<timestamp>/{rendered,backups,reports}
```

# Aruba CX Campus Automation – Excel Template Outline (v1)

This document defines the Excel workbook format used as the source of truth.

Design assumptions (v1)
- L2 campus (no EVPN)
- MST only
- All VLANs everywhere
- Core is VSX pair (model supplied in workbook)
- Access is 6200F/6300M stacks
- Gateway mode is site-level: CORE (SVIs on core) or FIREWALL (core mgmt SVI only)

## Workbook: customer-provided Excel file

## Tab: SITE
One row only.

Columns
- site_name
- timezone
- design_mode (use `L2` in v1)
- default_gateway_mode (CORE or FIREWALL)
- dns_servers (comma-separated)
- ntp_servers (comma-separated)
- syslog_servers (comma-separated, optional)
- mst_region_name
- mst_revision (integer)
- mgmt_vlan_id (integer)
- mgmt_vlan_name
- mgmt_svi_ip
- mgmt_svi_mask
- clearpass_enabled (Y/N)
- clearpass_servers (comma-separated, optional)

Example
- site_name: ExampleCampus
- default_gateway_mode: CORE
- dns_servers: 10.10.10.10,10.10.10.11
- mst_region_name: CUSTOMER-MST
- mgmt_vlan_id: 10

## Tab: DEVICES
One row per switch (core pair and each access stack member).

Columns
- device_name
- role (core|access)
- model (CORE|6200F|6300M)
- mgmt_ip
- mgmt_mask
- mgmt_gateway
- vsx_pair_id (for core)
- vsx_role (primary|secondary) (for core)
- stack_name (for access)
- stack_member_id (1,2,3...) (for access)

## Tab: VLANS
One row per VLAN.

Columns
- vlan_id
- vlan_name
- purpose (free text)
- enabled (Y/N)
- svi_gateway_ip (only used when default_gateway_mode=CORE)
- svi_mask
- dhcp_relay_ips (comma-separated, optional)

Notes
- In FIREWALL mode, svi_gateway_ip can be left blank.
- VLANs are created on all switches regardless.

## Tab: CORE_VSX
One row per VSX pair (usually 1 row per site).

Columns
- vsx_pair_id
- isl_port_1
- isl_port_2

## Tab: ACCESS_UPLINKS
One row per access stack.

Columns
- stack_name
- uplink_port_1 (default 1/1/51)
- uplink_port_2 (default 1/1/52)
- lacp_group_id
- core_peer_1
- core_peer_1_port
- core_peer_2
- core_peer_2_port

## Optional Tab: INTERFACE_DESCRIPTIONS
Columns
- device_name
- interface
- description

Used to override/standardize interface descriptions per customer.

## Strict v1 rules
- All enabled VLANs are deployed everywhere.
- Uplink trunks allow all enabled VLANs.
- MST config must match across all switches.
- CORE gateway mode: SVIs created on core for VLANs that specify svi_gateway_ip.
- FIREWALL gateway mode: core has ONLY mgmt SVI; no user SVIs.

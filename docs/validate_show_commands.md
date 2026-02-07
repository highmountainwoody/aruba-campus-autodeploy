# Aruba CX Campus – Useful Validation Commands (Engineer Quick Reference)

These are good manual checks to confirm the automation results.

Core (VSX pair)
- show vsx status
- show vsx brief
- show lacp interfaces
- show interfaces brief
- show vlan
- show ip interface brief
- show spanning-tree mst
- show spanning-tree mst configuration
- show running-config | include spanning-tree

Access (6200F/6300M stack)
- show vsf (if VSF stacking is used)
- show lacp interfaces
- show interfaces brief
- show vlan
- show spanning-tree mst

General troubleshooting
- show log -r
- show arp
- show mac-address-table

Notes
- Exact stacking command varies based on whether the access design uses VSF/stacking; validate.yml should use platform-appropriate commands.

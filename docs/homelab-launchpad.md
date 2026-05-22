# Homelab Launchpad

Quick links for day-to-day homelab operations. Public hostnames are protected with Cloudflare Access. Internal addresses require GlobalProtect, bastion, or the management workbench.

## Remote Entry Points

| Service | URL | Notes |
| --- | --- | --- |
| Management desktop | <https://mgmt.lanilsen.com> | Browser-based Linux desktop on `mgmt1`. Use this from phone or work laptop when you need an inside foothold. |
| Code workbench | <https://code.lanilsen.com> | code-server on `mgmt1`. Good for quick repo edits and terminal work. |
| Docs | <https://docs.lanilsen.com> | This documentation platform. |
| Grafana | <https://grafana.lanilsen.com> | Monitoring dashboards. |

## Internal Admin

| Service | URL | Notes |
| --- | --- | --- |
| Proxmox hp1 | <https://10.0.0.162:8006> | VLAN 16 Proxmox management. |
| Fortigate | <https://10.0.0.33:444> | Mo i Rana firewall management. |
| Palo Alto | <https://10.1.1.3> | Cabin firewall management. |
| MkDocs origin | <http://10.0.0.37> | Internal docs origin. |
| Monitoring origin | <http://10.0.0.38:3000> | Internal Grafana origin. |

## Out-of-Band

| Device | URL | Notes |
| --- | --- | --- |
| hp1 iLO | <https://10.0.124.164> | HP iLO management network. |
| hp2 iLO | <https://10.0.124.165> | HP iLO management network. |
| hp3 iLO | <https://10.0.124.163> | HP iLO management network. |
| Dell iDRAC | <https://10.0.124.162> | R820 out-of-band, not a primary cluster node. |

## Workbench Notes

`mgmt1` is intended to be the safe remote management surface:

- Cloudflare Access handles the public login.
- The web desktop provides Firefox and Remmina/FreeRDP for reaching internal web consoles and Windows RDP machines.
- code-server provides a persistent editor and terminal from inside the network.
- Terraform, Docker, Git, Ansible, and common network tools are installed by Ansible.

Keep direct management ports private. Prefer exposing new admin tools through Cloudflare Access or GlobalProtect rather than opening inbound firewall rules.

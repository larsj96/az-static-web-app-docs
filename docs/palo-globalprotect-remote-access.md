# Palo Alto GlobalProtect Remote Access

## Current State

GlobalProtect is published on the Palo Alto cabin firewall over Starlink IPv6.

```text
Hostname: gp.lanilsen.com
DNS: DNS-only Cloudflare AAAA
Current AAAA: 2a0d:3341:bb9c:af01::443
Portal: gp-cabin-portal
Gateway: gp-cabin-gateway
Listener interface: loopback.12
Listener zone: gp-public
Client tunnel: tunnel.12
Client zone: gp-clients
Client pool: 172.31.250.0/24
Client routes: 0.0.0.0/0 full tunnel
```

The trusted certificate is issued by Let's Encrypt and imported to PAN-OS as:

```text
gp-lanilsen-com
```

The certificate helper runs from the Frankfurt VPS as a Docker workflow in:

```text
/opt/homelab-ops/repos/terraform-palo/tools/letsencrypt-paloalto
```

## Terraform State

Palo Alto GlobalProtect state is stored in Cloudflare R2:

```text
Bucket: lanilsen-terraform-state
Key: terraform-palo/globalprotect/terraform.tfstate
```

Cloudflare DNS state is also in R2:

```text
Key: terraform-cloudflare/globalprotect-dns/terraform.tfstate
```

## Repos

```text
C:\Users\lars_\github\terraform-palo\live\homelab\globalprotect
C:\github\terraform-cloudflare\globalprotect-dns
```

## Apply Notes

The PAN-OS provider needs one helper object that is not modeled as a first-class resource:

```text
/network/tunnel/global-protect-gateway/entry[@name='tunnel.12']
```

This is handled by `terraform_data.globalprotect_network_tunnel`.

Important details:

- `gp-cabin-gateway` and `gp-cabin-portal` bind to `loopback.12` and exact IPv6 `2a0d:3341:bb9c:af01::443/128`.
- The network tunnel helper only sets `loopback.12`, `ip-address-family=ipv6`, and `tunnel-interface=tunnel.12`.
- Do not set the `/128` on both the provider gateway and the network tunnel helper, or PAN-OS rejects it as duplicate.
- Do not set a gateway-level IP pool on the network tunnel helper when the gateway client config already has `172.31.250.0/24`.

## Traffic Model

GlobalProtect clients connect over public IPv6, then use an IPv4 client pool inside the tunnel:

```text
phone/PC on internet
  -> gp.lanilsen.com AAAA
  -> Palo Alto loopback.12 / gp-public
  -> GlobalProtect tunnel.12 / gp-clients
  -> client IP from 172.31.250.0/24
```

Internet browsing should leave through the cabin Starlink WAN:

```text
172.31.250.0/24
  -> gp-clients-to-internet
  -> nat-gp-clients-to-starlink
  -> ethernet1/1 / wan-starlink
  -> Starlink internet
```

Mo i Rana access uses the existing Frankfurt hub chain:

```text
172.31.250.0/24
  -> gp-clients-to-vps-hub
  -> nat-gp-clients-to-vps-hub
  -> translated source 10.1.255.250
  -> Palo tunnel.10 / vpn-vps
  -> Frankfurt VPS
  -> VPS SNAT to 10.8.0.1
  -> Fortigate IPsec
  -> 10.0.0.0/16
```

The `nat-gp-clients-to-vps-hub` rule is important. The Palo-to-VPS and VPS-to-Fortigate selectors were built for `10.1.0.0/16`, `10.8.0.0/24`, and `10.0.0.0/16`, not for the GlobalProtect pool. Before this NAT existed, Palo traffic logs showed `172.31.250.1 -> 10.0.0.37` allowed by `gp-clients-to-vps-hub`, but the session aged out with `app=incomplete` because return traffic did not match the hub design.

## Validation

Known-good external test from the Frankfurt VPS:

```bash
curl -6 -k -i https://gp.lanilsen.com/ | sed -n '1,20p'
```

Expected:

```text
HTTP/1.1 302 Found
Location: /global-protect/login.esp
```

A curl from the workstation inside the same Palo Alto site may fail against the public IPv6 because it is a hairpin/same-firewall path. Use the VPS or another external IPv6 network for the public listener test.

Known-good client validation on `2026-05-22`:

```text
Android on 5G:
  GlobalProtect connected as lanilsen
  virtual IP 172.31.250.1
  reached mkdocs at 10.0.0.37 through Palo -> VPS -> Fortigate
```

Next normal tests:

```text
https://vg.no
http://10.0.0.37
https://10.0.0.33:444
```

Useful Palo log expectations:

```text
Internet:
  src 172.31.250.1
  rule gp-clients-to-internet
  to wan-starlink
  action allow

Mo i Rana:
  src 172.31.250.1
  dst 10.0.0.37 or 10.0.0.33
  rule gp-clients-to-vps-hub
  to vpn-vps
  action allow
```

## Authentication State

The portal and gateway are published, and a temporary PAN-OS local database user exists for bootstrap testing.

Current Terraform intent:

```text
authentication profile: gp-local-auth
auth backend: PAN-OS local database
bootstrap username: lanilsen
create_local_users: true, through private ignored tfvars
```

The private local file that preserves the bootstrap user across future applies is:

```text
C:\Users\lars_\github\terraform-palo\live\homelab\globalprotect\lanilsen-local-user.auto.tfvars
```

This file is ignored by git because it contains the temporary password. Do not commit it.

If the file is removed before SAML/MFA is ready, a normal Terraform apply will plan to remove the local `lanilsen` user. To recreate the bootstrap file, use this pattern:

```hcl
apply_when_certificate_exists = true
create_local_users            = true

local_users = {
  "lanilsen" = "REPLACE-WITH-PRIVATE-TEMP-PASSWORD"
}
```

Longer term, replace local database auth with SAML/MFA and then remove the bootstrap local user and private tfvars file.

## Remaining Work

- Prefer SAML/MFA after basic connectivity is stable.
- Test Windows and laptop GlobalProtect clients, not only Android.
- Decide whether to enable IPv6 inside the GlobalProtect client tunnel later. The current warning about `tunnel.12 ipv6 is not enabled` is not blocking the public IPv6 portal.

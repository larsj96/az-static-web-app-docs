# DNS And Naming Model

The homelab should use split DNS:

- Internal `nilsen-tech.com` sub-zones are authoritative on the Fortigate side.
- Palo Alto and GlobalProtect clients reuse those Fortigate zones by conditional forwarding over the site-to-site tunnel.
- Public `lanilsen.com` names are managed in Cloudflare and usually point to Cloudflare Tunnel.

This keeps internal names useful from the cabin, GlobalProtect, and Mo i Rana without publishing private IPs to public DNS.

## Internal Zones

Fortigate owns the internal recursive/authoritative zones:

```text
mgmt.nilsen-tech.com
ilo.nilsen-tech.com
```

Terraform stack:

```text
C:\github\terraform-fortigate\internal-dns-vip
```

State key:

```text
terraform-fortigate/internal-dns-vip/terraform.tfstate
```

The Fortigate DNS stack reads Proxmox remote state from Cloudflare R2 and creates records for Terraform-managed VMs and out-of-band management endpoints.

Important records:

| Name | Target | Purpose |
| --- | --- | --- |
| `proxmox1.mgmt.nilsen-tech.com` | `10.0.0.162` | hp1 routed Proxmox management |
| `docs.mgmt.nilsen-tech.com` | `10.0.0.35` | MkDocs VM |
| `auth1.mgmt.nilsen-tech.com` | `10.0.0.36` | Authentik |
| `docker1.mgmt.nilsen-tech.com` | `10.0.0.37` | Docker host |
| `grafana.mgmt.nilsen-tech.com` | `10.0.0.38` | Grafana/monitoring |
| `media1.mgmt.nilsen-tech.com` | `10.0.0.39` | Media stack VM |
| `plex1.mgmt.nilsen-tech.com` | `10.0.0.39` | Friendly Plex/media alias |
| `hp1.ilo.nilsen-tech.com` | `10.0.124.164` | hp1 iLO |
| `hp2.ilo.nilsen-tech.com` | `10.0.124.165` | hp2 iLO |
| `hp3.ilo.nilsen-tech.com` | `10.0.124.163` | hp3 iLO |
| `dell1.ilo.nilsen-tech.com` | `10.0.124.162` | Dell iDRAC |

## Palo Alto And GlobalProtect

Palo Alto should not maintain a second copy of Fortigate records.

Desired resolver flow:

```text
GlobalProtect client or Palo-side LAN client
  -> Palo DNS proxy
  -> conditional forward mgmt.nilsen-tech.com to 10.0.0.33
  -> conditional forward ilo.nilsen-tech.com to 10.0.0.33
  -> Fortigate DNS zone answer
```

For everything else, Palo can use normal public DNS resolvers.

This depends on the Palo-to-Fortigate tunnel being up:

```text
Palo 10.1.0.0/16 <-> Fortigate 10.0.0.0/16
```

The DNS traffic itself is ordinary IPv4 inside the tunnel:

```text
Palo DNS proxy
  source: Palo-side interface / 10.1.0.0/16
  destination: Fortigate DNS / 10.0.0.33
  transport path: IPv6 IPsec tunnel.20 when healthy
```

That is the important bit: DNS does not need native IPv6 on the inside networks yet. The direct Palo-to-Fortigate tunnel uses IPv6 as the outer transport, and it carries the IPv4 DNS queries to the Fortigate DNS service.

Route preference should make the DNS path automatic:

```text
10.0.0.0/16 via tunnel.20 direct IPv6 IPsec, metric 5
10.0.0.0/16 via tunnel.10 Frankfurt VPS fallback, metric 50
```

If the direct IPv6 tunnel is up, DNS follows the preferred direct path. If the direct path is down, routing can fall back through the Frankfurt VPS where configured.

Terraform note:

```text
C:\github\terraform-palo\live\homelab\network-base\dns-proxy.tf.example
```

That file documents the intended DNS proxy and conditional forwarding behavior. The active provider schema for DNS proxy must be validated before turning it into live state.

## Public Cloudflare Names

Public names under `lanilsen.com` live in Cloudflare.

Terraform stack:

```text
C:\github\terraform-cloudflare\docs-tunnel
```

Public services currently follow this pattern:

| Public name | Origin |
| --- | --- |
| `docs.lanilsen.com` | `http://10.0.0.35` |
| `grafana.lanilsen.com` | `http://10.0.0.38:3000` |
| `auth.lanilsen.com` | `http://10.0.0.36:9000` |
| `plex1.lanilsen.com` | `http://plex1.mgmt.nilsen-tech.com:32400` |

Cloudflare Tunnel config now supports `public_tunnel_apps` for extra names:

```hcl
public_tunnel_apps = {
  plex1 = {
    hostname       = "plex1.lanilsen.com"
    origin_url     = "http://plex1.mgmt.nilsen-tech.com:32400"
    access_enabled = false
  }
}
```

Use Cloudflare Access for admin web apps. Be careful with Plex: Cloudflare Tunnel is fine for light web access, but heavy streaming should prefer GlobalProtect/VPN or a direct remote-access design.

`plex1` currently points straight at Plex on port `32400` because the internal SSL reverse proxy is not deployed yet. After the reverse proxy exists and has trusted certificates, change the Cloudflare origin to `https://plex1.mgmt.nilsen-tech.com`.

## Reverse Proxy Direction

Internal HTTPS should be handled by a reverse proxy, not by one Fortigate VIP per app.

Preferred model:

```text
Fortigate DNS
  -> plex1.mgmt.nilsen-tech.com
  -> internal reverse proxy :443
  -> Plex backend on media1
```

Certificate automation should use Cloudflare DNS-01 where possible so the reverse proxy can issue trusted certificates without exposing internal apps directly.

## Validation

From a Fortigate-side VM:

```bash
dig @10.0.0.33 plex1.mgmt.nilsen-tech.com
dig @10.0.0.33 hp1.ilo.nilsen-tech.com
```

From a Palo-side client or GlobalProtect client after DNS proxy forwarding is enabled:

```bash
nslookup plex1.mgmt.nilsen-tech.com
nslookup hp1.ilo.nilsen-tech.com
```

Expected:

```text
plex1.mgmt.nilsen-tech.com -> 10.0.0.39
hp1.ilo.nilsen-tech.com    -> 10.0.124.164
```

For public Cloudflare names:

```bash
nslookup plex1.lanilsen.com
curl -I https://plex1.lanilsen.com
```

Expected DNS answer is a proxied Cloudflare response, not the private homelab IP.

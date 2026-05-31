# CS2 Scrim Server

## Purpose

Run a low-friction Counter-Strike 2 scrim/practice server from the Frankfurt VPS hub while keeping the setup repeatable in Ansible and observable from the homelab monitoring stack.

The VPS-hosted server is the preferred public scrim location because it avoids adding Starlink and site-to-site IPsec jitter into player traffic.

## Live Server

| Item | Value |
| --- | --- |
| Host | `vpshub` |
| Public IP | `72.61.95.150` |
| Game DNS | `game.cs2.lanilsen.com` |
| Stack path | `/opt/homelab-ops/game-servers/cs2-test` |
| Container | `cs2-test` |
| Game port | `27016/udp` |
| RCON helper | `/usr/local/bin/cs2-rcon` |
| Map helper | `/usr/local/bin/cs2-map` |
| Base image | `xbird/cs2-matchzy:latest-pelican` |
| Plugins | CounterStrikeSharp, MatchZy, ChangeLevelChat |

The older `27015` test server was removed. Treat `27016` as the active scrim/practice server.

Players should connect to:

```text
connect game.cs2.lanilsen.com:27016
```

`game.cs2.lanilsen.com` is intentionally DNS-only in Cloudflare. Do not proxy the game endpoint; Cloudflare's orange-cloud proxy is for HTTP(S), not CS2 UDP gameplay.

## Get5 Web Panel

Get5/G5V is hosted on-prem on `docker1` and published through the existing homelab Cloudflare Tunnel:

| Item | Value |
| --- | --- |
| Public URL | `https://cs2.lanilsen.com` |
| Access policy | Cloudflare Access one-time PIN |
| Allowed users | `larsj96@gmail.com`, `mikael.fjell@hotmail.com` |
| Origin host | `docker1` / `10.0.0.37` |
| Origin URL | `http://10.0.0.37:33080` |
| Stack path | `/opt/get5-panel` |
| Containers | `get5-nginx`, `get5-web`, `get5-api`, `get5db`, `get5-redis` |

This split is deliberate:

- `cs2.lanilsen.com` is the protected HTTPS web panel.
- `game.cs2.lanilsen.com:27016` is the direct game server endpoint.

Do not collapse these into one hostname unless the Cloudflare design changes, because the web panel must be proxied and Access-protected while the game port must remain direct.

Operational checks from `docker1`:

```bash
cd /opt/get5-panel
docker compose ps
curl -I http://127.0.0.1:33080/
curl -I http://127.0.0.1:33080/api-docs/
```

The stack currently permits local logins. A Steam Web API key can be added later for richer Steam lookups; store it in Vault and render it into `/opt/get5-panel/.env` rather than committing it to Git.

## Repeatable Configuration

Ansible owns the helper layer:

```text
ansible-homelab/playbooks/cs2.yml
ansible-homelab/group_vars/cs2_servers.yml
ansible-homelab/roles/cs2_server/
```

Run from the workstation or another Ansible control host that can SSH to the VPS:

```bash
cd /mnt/c/github/ansible-homelab
ANSIBLE_ROLES_PATH=/mnt/c/github/ansible-homelab/roles \
  ansible-playbook -i inventory/homelab.ini -b playbooks/cs2.yml --limit vpshub
```

The playbook installs:

- `cs2-map`, a map alias helper.
- `cs2-rcon`, a safe wrapper around the local RCON password in the stack `.env`.
- `maps.tsv`, the source of truth for map aliases.
- The ChangeLevelChat map allow list, when the plugin config directory exists.

## Map Commands

Check available aliases:

```bash
cs2-map list
```

Change to an official/local map:

```bash
cs2-map cache
cs2-map mirage
```

Load a Workshop map:

```bash
cs2-map redline
cs2-map rats
cs2-map freemirage
cs2-map dusthr
cs2-map hoejhus
```

Current managed Workshop maps:

| Alias | Workshop ID | Map name |
| --- | --- | --- |
| `redline` | `3199551320` | `aim_redline` |
| `rats` | `3072959547` | `de_rats_1337` |
| `freemirage` | `3298850698` | `Free Mirage` |
| `dusthr` | `3127729110` | `de_dust_hr` |
| `hoejhus` | `3439554221` | `hoejhus9` |

Current managed local/official maps:

| Alias | Map |
| --- | --- |
| `cache` | `de_cache` |
| `mirage` | `de_mirage` |
| `dust2` | `de_dust2` |
| `inferno` | `de_inferno` |
| `nuke` | `de_nuke` |
| `ancient` | `de_ancient` |
| `anubis` | `de_anubis` |
| `train` | `de_train` |

`de_cache.vpk` and `de_cache_vanity.vpk` were copied from the local Steam install into the server map directory:

```text
/opt/homelab-ops/game-servers/cs2-test/cs2-data/game/csgo/maps/
```

## In-Game Map Change

ChangeLevelChat is installed so players can use chat commands such as:

```text
!maps
!changelevel de_cache
!changelevel de_dust_hr
```

The current plugin is a direct change command, not a vote system. Keep that in mind if the server is public or semi-public. `!changetoggle` is admin-only; the basic map change command is available to players.

If the plugin does not pick up a newly rendered map list immediately, restart the CS2 container during a quiet window:

```bash
cd /opt/homelab-ops/game-servers/cs2-test
docker compose restart cs2-test
```

## RCON Operations

Check server status:

```bash
cs2-rcon status
```

Run a raw command:

```bash
cs2-rcon "css_plugins list"
```

The RCON password is read from the stack `.env` on the VPS. Do not copy it into documentation, shell history, or Git.

## Monitoring And Logs

The VPS hub is monitored as `vpshub`, not as a single-purpose `cs2scrim` host.

Ansible inventory places `vpshub` in:

```text
cs2_servers
telegraf_agents
log_shipper_agents
```

Telegraf collects host, Docker, container CPU, container memory, container network, disk, disk I/O, and route-sensitive IPsec host metrics. The Grafana CS2 dashboard filters on:

```text
host = vpshub
container_name = cs2-test
```

Filebeat ships VPS logs into the shared OpenSearch/ELK pipeline. Use those logs together with `docker logs cs2-test` when players report server FPS warnings or plugin issues.

Useful live checks:

```bash
docker stats --no-stream cs2-test
docker logs --tail 120 cs2-test
systemctl status telegraf
ip route get 10.0.0.38
```

Expected monitoring route from the VPS to `monitoring1`:

```text
10.0.0.38 via 72.61.95.254 dev eth0 src 10.8.0.1
```

That source address matters because the Fortigate IPsec selector expects the VPS transit subnet. If metrics stop arriving from `vpshub`, check `homelab-ipsec-routes.service` before blaming Telegraf.

## Performance Notes

Observed with three connected players:

- Player ping looked normal around 30-40 ms.
- Host CPU and memory had plenty of headroom.
- CS2 logs showed occasional `UNEXPECTED LONG FRAME DETECTED` and Steam networking lock warnings.

If those warnings continue, the next low-risk tuning step is to test the CS2 container with Docker host networking during a maintenance window. That removes Docker NAT/proxy from the game path, but requires a container restart.

Do not move the primary scrim server behind the homelab Fortigate unless there is a deliberate reason. A homelab-hosted public CS2 server can be NATed through the VPS, but it adds Starlink/IPsec jitter and should be treated as a secondary lab server.

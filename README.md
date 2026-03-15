# NetworkOptimizer for UniFi

Self-hosted performance optimization and security audit tool for UniFi Networks, deployed via Docker on this host.

> Source: [Ozark-Connect/NetworkOptimizer](https://github.com/Ozark-Connect/NetworkOptimizer) — Business Source License 1.1. Free for personal/non-commercial use (up to 3 sites). Converts to Apache 2.0 on January 1, 2028.

## Features

- **Wi-Fi Optimization** — Wireless health scoring: site health, RF environment, channel recommendations, client stats, roaming, and airtime fairness (12 analysis tabs).
- **Security Auditing** — 63 checks across 5 categories (firewall, VLAN isolation, DNS, port security, UPnP), scored 0–100.
- **Performance Testing** — WAN speed (Cloudflare), LAN speed (iperf3), browser-based client tests with network path visualization.
- **Advanced Monitoring** — Adaptive SQM for bufferbloat, cellular modem stats (U5G-Max), UPnP inspection, threat intelligence with IPS correlation.
- **Scheduling & Alerts** — Email, Discord, Slack, Microsoft Teams, and webhooks.

## This Deployment

| Item | Value |
|---|---|
| Main App | https://optimizer.mgsfoxhound.net |
| Speed Test | https://speedtest.mgsfoxhound.net |
| Internal Port | 8042 (localhost only — not directly accessible) |
| Reverse Proxy | Nginx Proxy Manager |
| Timezone | America/Chicago |

## Requirements

- Docker 20.10+ and Docker Compose 2.0+
- 2 GB RAM minimum (4 GB recommended)
- 10 GB disk space

## Initial Setup

1. Configure environment variables:
   ```bash
   cp .env.example .env
   nano .env
   ```

2. Start the stack:
   ```bash
   docker compose up -d
   ```

3. Verify:
   ```bash
   docker compose ps
   curl http://localhost:8042/api/health
   ```
   Expected: `network-optimizer` shows `Up (healthy)`.

4. Access the web UI at https://optimizer.mgsfoxhound.net

### Admin Password

On first run the password is auto-generated and shown in logs:

```bash
docker logs network-optimizer 2>&1 | grep -A5 "AUTO-GENERATED"
```

Set a permanent password in **Settings > Admin Password** (recommended), or pre-set via `APP_PASSWORD` in `.env` before starting.

> **Password precedence:** Database (Settings UI) > `APP_PASSWORD` env var > Auto-generated

## Reverse Proxy — Nginx Proxy Manager

This deployment uses Nginx Proxy Manager for HTTPS termination with Let's Encrypt.

### Main App — optimizer.mgsfoxhound.net → localhost:8042

Create a Proxy Host in NPM:

| Setting | Value |
|---|---|
| Domain Names | `optimizer.mgsfoxhound.net` |
| Forward Hostname/IP | `127.0.0.1` |
| Forward Port | `8042` |
| Websockets Support | **Enabled** (required for Blazor) |
| SSL | Let's Encrypt, Force HTTPS |

### Speed Test — speedtest.mgsfoxhound.net → localhost:3005

> ⚠️ The speedtest proxy **must force HTTP/1.1**. HTTP/2 multiplexing artificially inflates results.
> The main app requires HTTP/2+ for Blazor WebSockets — these **must be separate proxy hosts**.

Create a second Proxy Host:

| Setting | Value |
|---|---|
| Domain Names | `speedtest.mgsfoxhound.net` |
| Forward Hostname/IP | `127.0.0.1` |
| Forward Port | `3005` |
| Websockets Support | Off |
| SSL | Let's Encrypt, Force HTTPS |

In the **Advanced** tab, add the following custom nginx config to force HTTP/1.1:

```nginx
proxy_http_version 1.1;
proxy_set_header Upgrade $http_upgrade;
proxy_set_header Connection "";

proxy_buffering on;
proxy_buffer_size 128k;
proxy_buffers 4 256k;
proxy_busy_buffers_size 256k;
```

### HTTPS + Geolocation

With `OPENSPEEDTEST_HTTPS=true` and `REVERSE_PROXIED_HOST_NAME` configured, browser geolocation is available during speed tests — useful for mapping Wi-Fi dead zones or verifying coverage across a property.

## Ports

| Port | Service | Externally Exposed |
|---|---|---|
| 8042 | Main web UI | No — localhost only (`BIND_LOCALHOST_ONLY=true`) |
| 3005 | OpenSpeedTest | Yes — via NPM at speedtest.mgsfoxhound.net |
| 5201 | iperf3 | Only if `IPERF3_SERVER_ENABLED=true` in `.env` |

## Service Management

```bash
# View live logs (all services)
docker compose logs -f

# View logs for main app only
docker compose logs -f network-optimizer

# Restart services
docker compose restart

# Stop all services
docker compose down

# Upgrade to latest images
docker compose pull && docker compose up -d

# Reload env var changes (restart alone does NOT reload .env)
docker compose down && docker compose up -d
```

## SSH Configuration (SQM & LAN Speed Tests)

SSH access to UniFi devices is required for the following features:

| Feature | Gateway SSH | Device SSH |
|---|---|---|
| Adaptive SQM | Required | — |
| WAN Speed Test (gateway-based) | Required | — |
| LAN Speed Test (gateway) | Required | — |
| LAN Speed Test (devices) | — | Required |
| Security Audit | — | — |
| Wi-Fi Optimizer | — | — |

**Enable Gateway SSH:** UniFi > Settings > Control Plane > Console > Enable SSH. Username: `root`.

**Enable Device SSH (UniFi Network 9.5+):** UniFi Devices > Device Updates and Settings > Device SSH Settings > Enable Device SSH Authentication.

Place SSH private keys in the `./ssh-keys/` directory (mounted read-only into the container):

- `./ssh-keys/gateway_key` — UniFi gateway/console
- `./ssh-keys/id_rsa` — UniFi network devices

Configure credentials in the app under **Settings > Gateway SSH** and **Settings > Device SSH**.

## UniFi Account

Use a dedicated **Local Access Only** account — Ubiquiti SSO accounts will not work.

Recommended permissions:
- Super Admin, or
- Restricted: View Only for Network and Protect; None for User & Account Management

## Backup

```bash
# Manual backup
tar czf /backups/network-optimizer-$(date +%Y%m%d-%H%M%S).tar.gz -C /opt/network-optimizer data/

# Restore
docker compose down
tar xzf /backups/network-optimizer-YYYYMMDD-HHMMSS.tar.gz -C /opt/network-optimizer/
docker compose up -d
```

Automated daily backup — add via `crontab -e`:

```
0 2 * * * tar czf /backups/network-optimizer/data-$(date +\%Y\%m\%d-\%H\%M\%S).tar.gz -C /opt/network-optimizer data/ && find /backups/network-optimizer -type f -mtime +7 -delete
```

## Security

```bash
chmod 600 .env
chmod 700 data/
```

- `BIND_LOCALHOST_ONLY=true` is set — port 8042 is not accessible from the network directly
- All access is routed through Nginx Proxy Manager with HTTPS
- Use a dedicated UniFi Local Access Only account with minimum required permissions

## Firewall (UFW)

```bash
sudo ufw allow 22/tcp
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw enable
# Port 8042 does not need to be open (BIND_LOCALHOST_ONLY=true)
```

## Upgrade

```bash
docker compose pull
docker compose up -d
```

## Troubleshooting

**Get auto-generated admin password:**
```bash
docker logs network-optimizer 2>&1 | grep -A5 "AUTO-GENERATED"
```

**Reset admin password:**
```bash
curl -fsSL https://raw.githubusercontent.com/Ozark-Connect/NetworkOptimizer/main/scripts/reset-password.sh | bash
```

**Container won't start:**
```bash
docker compose logs network-optimizer
```
Common causes: port 8042 in use, permission denied on `data/`, out of disk space.

**Can't connect to UniFi:**
- Use a Local Access Only account (not SSO)
- Enable "Ignore SSL errors" in app settings if controller uses a self-signed cert
- Test connectivity from inside the container: `curl -k https://your-controller:443`

**Env var changes not taking effect:**
`docker compose restart` does not reload env vars. Use:
```bash
docker compose down && docker compose up -d
```

**Blazor UI not loading via NPM:**
- Verify **Websockets Support** is enabled on the main app proxy host in NPM
- Ensure proxy timeout is 60s or higher in NPM (Advanced tab: `proxy_read_timeout 60s;`)

**SSH errors in security audit / SQM:**
Test SSH connectivity from inside the container:
```bash
docker exec -it network-optimizer ssh root@your-gateway-ip
```
Check for CyberSecure IPS rules that may block outbound SSH scans.

## Support

- [GitHub Issues](https://github.com/Ozark-Connect/NetworkOptimizer/issues)
- Email: tj@ozarkconnect.net
- License: Business Source License 1.1 — free for personal/non-commercial use (up to 3 sites). Converts to Apache 2.0 on January 1, 2028.

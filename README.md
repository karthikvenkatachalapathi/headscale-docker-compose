# Self-Hosted Headscale VPN Stack

A Docker Compose deployment of **Headscale**, **Headplane**, **Nginx Proxy Manager**, and **DuckDNS**, with an optional Prometheus-compatible metrics exporter.

This is a reusable reference deployment. Replace all example domains, paths, and secrets before using it. **The Compose file alone is not sufficient:** Headscale and Headplane each require their own configuration files.

## Architecture

```text
Tailscale clients
      |
      | HTTPS :443
      v
Public DNS (headscale.example.com)
      |
      v
Router/firewall -> Nginx Proxy Manager -> Headscale :8080
                    |
                    +----------------------> Headplane :3000
                       (headscale-ui.example.com)

DuckDNS -> updates a DuckDNS hostname to the host's public IP
Headplane -> Headscale API / shared configuration / optional Docker integration
```

Headscale is the control plane; normal tailnet traffic generally flows directly between clients, with DERP relays used when necessary.

## Components

| Service | Image | Role | Default host port |
| --- | --- | --- | --- |
| Headscale | `headscale/headscale` | Tailscale-compatible control server | `8080/tcp` |
| Headplane | `ghcr.io/tale/headplane` | Headscale administration UI | `3303/tcp` (maps to container `3000`) |
| Nginx Proxy Manager | `jc21/nginx-proxy-manager` | Public HTTPS reverse proxy and certificate management | `80`, `443`, `81/tcp` |
| DuckDNS | `lscr.io/linuxserver/duckdns` | Updates dynamic DNS | None |
| Tailscale exporter (optional) | `adinhodovic/tailscale-exporter` | Prometheus monitoring | `9250/tcp` when enabled |

You can omit DuckDNS if your domain already uses an alternative dynamic DNS solution, and omit Nginx Proxy Manager if you already have a compatible reverse proxy.

## Prerequisites

- Linux server with Docker Engine and Docker Compose v2 (`docker compose`).
- A public DNS name you control, such as `headscale.example.com`.
- The ability to forward public TCP port **443** to the reverse proxy; port **80** may also be needed for HTTP certificate validation.
- A current, compatible Headscale version (Headplane's Docker documentation currently requires Headscale **0.27.0+**).
- Persistent local storage and secure backups.
- Optional: DuckDNS account/token and a hostname if your public IP changes.

> **Important:** Headscale's protocol must work through the chosen reverse proxy. Do **not** put the Headscale control endpoint behind a Cloudflare Tunnel or Cloudflare's orange-cloud HTTP proxy. If using Cloudflare DNS, set the Headscale record to **DNS only**. This restriction does not automatically apply to unrelated applications.

## Repository layout

The supplied Compose configuration uses `${HOME_DIR}/docker/headscale/...` for persistent paths:

```text
project/
├── compose.yaml
├── .env                    # private; do not commit
└── README.md

${HOME_DIR}/docker/headscale/
├── headscale-config/
│   ├── config.yaml         # private / instance-specific
│   └── dns_records.json    # optional, for extra DNS records
├── headscale-data/
├── headplane-config/
│   ├── config.yaml         # private / instance-specific
│   └── headplane-data/
├── nginx-proxy/
│   ├── data/
│   └── letsencrypt/
└── duckdns/
    └── config/
```

Set `HOME_DIR` to an **absolute path**, such as `/opt`, not a relative directory. Files mounted as individual files (especially `config.yaml` and `dns_records.json`) must exist as files before you start the stack. Otherwise, Docker can create directories in their place.

## 1. Prepare the host

Copy the shared, sanitized Compose YAML into `compose.yaml`. Then create the external network referenced by the file:

```bash
docker network create headscale-network
```

If it already exists, reuse it. To create the persistent structure (assuming `HOME_DIR=/opt`):

```bash
export HOME_DIR=/opt
mkdir -p "$HOME_DIR/docker/headscale/"{headscale-config,headscale-data,headplane-config/headplane-data,nginx-proxy/data,nginx-proxy/letsencrypt,duckdns/config}
touch "$HOME_DIR/docker/headscale/headscale-config/dns_records.json"
printf '[]\n' > "$HOME_DIR/docker/headscale/headscale-config/dns_records.json"
```

The `dns_records.json` file is relevant only when your Headscale DNS configuration uses an extra-records file. If not using that feature, remove its individual Headplane mount.

## 2. Configure environment variables

Create a `.env` file alongside `compose.yaml`:

```dotenv
# Absolute base directory for the mounted persistent data
HOME_DIR=/opt

# Only needed if you keep the DuckDNS service
DUCKDNS_SUBDOMAIN=your-subdomain
DUCKDNS_TOKEN=replace-with-your-private-token

# Only needed if you enable the optional exporter
HEADSCALE_API_KEY=replace-with-your-api-key
```

Keep `.env` private. Add `.env`, backups, API keys, and data/config directories containing secrets to `.gitignore`. Do not commit real passwords, secrets, private keys, authentication cookies, or Headscale databases to a public repository.

## 3. Configure Headscale

Download the **sample configuration matching the exact Headscale release you deploy** from the [Headscale repository](https://github.com/juanfont/headscale) or consult the [official configuration documentation](https://headscale.net/stable/ref/configuration/).

Save the completed file at:

```text
${HOME_DIR}/docker/headscale/headscale-config/config.yaml
```

Check at least these settings (the exact fields/defaults can vary by release):

- `server_url`: public HTTPS origin, for example `https://headscale.example.com`.
- `listen_addr`: must listen on an address reachable by the NPM container, generally `0.0.0.0:8080` **inside the container**. A `127.0.0.1` listener inside the Headscale container will not be reachable from a separate NPM container.
- TLS certificate/key fields: typically left empty if NPM terminates HTTPS.
- Data paths: persist database and cryptographic state under `/var/lib/headscale`.
- DNS/MagicDNS: choose your tailnet domain, nameservers, and optional `dns.extra_records_path`.
- DERP: decide whether to use public DERP servers or enable the Headscale embedded DERP server.
- Trusted proxies: only allow the actual proxy network/CIDR, and configure the proxy to sanitize forwarding headers.

**If using embedded DERP/STUN**, expose and forward `3478/udp` to Headscale as required by your config. The supplied Compose example does not publish that port, so add it explicitly if needed:

```yaml
# Under services.headscale.ports:
- "3478:3478/udp"
```

Do not expose Headscale gRPC management (`50443`) to the public internet. It is optional for this setup. The reference Compose binds it to loopback; if it is unused, remove that port mapping entirely.

## 4. Configure Headplane

Use the [official Headplane configuration example](https://github.com/tale/headplane/blob/main/config.example.yaml) for the **version you deploy**, then save the completed configuration to:

```text
${HOME_DIR}/docker/headscale/headplane-config/config.yaml
```

Settings to review:

- Headscale endpoint: use `http://headscale:8080` for internal container-to-container access where supported.
- Headscale configuration path: `/etc/headscale/config.yaml`.
- Optional DNS records path: `/etc/headscale/dns_records.json` if configured.
- Persistent Headplane data path: `/var/lib/headplane`.
- Cookie secret: generate a unique secret, for example `openssl rand -base64 24`.
- Docker integration: enable it only if you want Headplane to manage/restart the Headscale container and understand its security implications.

**Security caution:** mounting `/var/run/docker.sock:ro` does **not** make the Docker API read-only. Anyone who can access that socket may have powerful control over the Docker host. For a hardened installation, use a restricted Docker socket proxy or disable the integration if you do not need it.

The Headscale configuration and DNS records mounts in the reference Compose are **read-write** because Headplane's advanced network management may edit them. Use read-only mounts if you do not want the web UI to change those files (and disable associated write features).

## 5. Set up DNS and public reachability

Create DNS records for your chosen hostnames:

| Hostname | Destination | Purpose |
| --- | --- | --- |
| `headscale.example.com` | Your public IP or a properly configured CNAME | Headscale control plane |
| `headscale-ui.example.com` | Your public IP, or internal DNS if UI is private | Headplane UI |

If your ISP assigns a changing public IP, configure DuckDNS and point an appropriate hostname at it, or use a different dynamic DNS updater. DuckDNS is **not** automatically linked to arbitrary custom domains; configure your DNS provider's records as appropriate.

### Router / firewall port-forwarding (required)

Create a **WAN port-forward / destination NAT rule** on your router or firewall:

| Setting | Value |
| --- | --- |
| Name | `Headscale HTTPS` |
| Incoming interface | `WAN` |
| Protocol | **TCP only** (not UDP) |
| External / WAN port | **443** |
| Destination IP | **LAN IP address of the Docker host** running Nginx Proxy Manager (e.g. `192.168.1.50`) |
| Internal / destination port | **443** |
| Action | Allow / Forward |

Traffic path: `Internet → WAN TCP/443 → Docker host LAN IP TCP/443 → NPM container TCP/443 → Headscale container TCP/8080`.

**Important:** With the provided Compose mapping `"443:443"`, forward to the **Docker host's LAN IP**, not the container's private Docker bridge IP. Docker handles the host-to-container port mapping. If you deploy NPM using macvlan or another design where it has its own routable LAN IP, forward to that IP instead.

Only **TCP 443** is needed for the public HTTPS entry point described here. You do **not** need to open UDP 443. Optionally forward **TCP 80** for HTTP-01 certificate validation or HTTP-to-HTTPS redirect; DNS-01 certificate issuance avoids that requirement. If you separately enable Headscale's embedded DERP/STUN service, its **UDP 3478** requirement is a different optional rule (see the Headscale configuration section).

Do **not** forward NPM's administration port (`81`), Headplane's port (`3303`), or Headscale's management ports publicly.

> If your ISP uses CGNAT or blocks incoming ports, standard port forwarding may not work. You'll need a reachable public endpoint and a supported networking/reverse-proxy design.

## 6. Configure Nginx Proxy Manager

After deployment, open the NPM admin UI from a trusted network at `http://SERVER_IP:81`, change the default credentials, and secure administrative access. Create proxy hosts:

| Domain | Scheme | Forward host | Forward port |
| --- | --- | --- | --- |
| `headscale.example.com` | HTTP | `headscale` | `8080` |
| `headscale-ui.example.com` | HTTP | `headplane` | `3000` |

For each hostname, issue a valid SSL certificate and enable HTTPS. Headscale's proxy path must support the Tailscale control protocol upgrade and avoid request/response buffering where the Headscale reverse-proxy documentation requires it. **NPM's standard checkbox/settings alone should not be assumed sufficient**: test actual client registration and connectivity, and consult the [official Headscale reverse-proxy guide](https://headscale.net/stable/ref/integration/reverse-proxy/).

Keep the Headplane admin UI private if possible (VPN-only or otherwise access-restricted). Separate domains, as used here, are an alternative to Headplane's single-domain `/admin` integration; follow the Headplane docs for any cross-origin or authentication settings required by your version.

## 7. Validate and launch

From the directory containing `compose.yaml`:

```bash
# Confirm substitutions, syntax and merged Compose settings
docker compose config

# Start containers
docker compose up -d

# Check health/status
docker compose ps

# Inspect logs
docker compose logs -f headscale headplane npm-headscale
```

Look for startup failures due to missing config files, invalid YAML, container permissions, mismatched configuration paths, or unavailable ports.

## 8. Create a Headscale user and authenticate Headplane

Create a tailnet user:

```bash
docker exec -it headscale headscale users create example-user
```

Create an API key for Headplane login (treat the output like a password):

```bash
docker exec -it headscale headscale apikeys create --expiration 90d
```

Open `https://headscale-ui.example.com/admin` (or `http://SERVER_IP:3303/admin` from a trusted network before reverse-proxy setup), then authenticate using the API key as directed by the Headplane version in use.

To connect a client with Tailscale installed, point it at your Headscale login server:

```bash
tailscale up --login-server https://headscale.example.com
```

Complete the registration process shown by the client. Depending on the Headscale release and your chosen registration method, you may need to register a node from the Headscale CLI or create and use a pre-authentication key. See the [Headscale getting-started guide](https://headscale.net/stable/setup/).

## 9. Optional monitoring

The commented exporter service can be enabled if you have a Prometheus/Grafana monitoring stack and have confirmed that the exporter works with your Headscale release. Create a scoped Headscale API key, keep the endpoint bound to a trusted interface, and avoid exposing Prometheus metrics publicly.

## Troubleshooting

| Problem | What to check |
| --- | --- |
| Headplane fails on startup | Config file exists, configuration matches release, cookie secret, log output |
| Headplane cannot connect to Headscale | Docker network, `http://headscale:8080`, API authentication, service availability |
| Headplane cannot edit Headscale settings | Shared files are writable, paths match, integration enabled, Docker socket permissions |
| NPM returns `502 Bad Gateway` | `headscale`/`headplane` service names, container readiness, correct ports, shared network |
| Device registration fails | Public DNS, valid TLS certificate, Headscale `server_url`, reverse-proxy protocol upgrades |
| Device connects but is slow or relayed | NAT traversal, UDP connectivity, DERP/STUN setup, client diagnostics |
| SSL certificate fails | DNS points to correct endpoint, public 80/443 reachability or valid DNS challenge |
| DuckDNS doesn't update | Token, subdomain spelling, IP detection, container logs |
| `config.yaml` unexpectedly becomes a directory | Missing host-side file at bind-mount time; correct path and recreate file |

Useful commands:

```bash
docker compose ps
docker compose logs --tail=100 headscale
docker compose logs --tail=100 headplane
docker compose logs --tail=100 npm-headscale
docker exec -it headscale headscale users list
docker exec -it headscale headscale nodes list
```

## Security and maintenance checklist

- [ ] Use pinned, tested image versions rather than floating `latest` tags in production.
- [ ] Protect `.env`, Headscale keys, and both applications' configuration/state files.
- [ ] Limit NPM management (`81`) and Headplane management (`3303`) to trusted networks.
- [ ] Do not publicly expose Docker socket, Headscale gRPC, or metrics endpoints.
- [ ] Keep Headscale control hostname on a supported direct HTTPS/reverse-proxy path.
- [ ] Ensure HTTPS certificates renew correctly.
- [ ] Back up **Headscale state/config** and **Headplane state/config** before upgrades.
- [ ] Test actual tailnet client registration, DNS resolution, and connectivity.
- [ ] Rotate API credentials and revoke unnecessary pre-authentication keys.
- [ ] Review release notes and verify compatibility before image upgrades.

## Official references

- [Headscale documentation](https://headscale.net/)
- [Headscale reverse proxy](https://headscale.net/stable/ref/integration/reverse-proxy/)
- [Headplane Docker installation](https://headplane.net/install/docker)
- [Headplane configuration example](https://github.com/tale/headplane/blob/main/config.example.yaml)
- [Nginx Proxy Manager](https://nginxproxymanager.com/)
- [DuckDNS](https://www.duckdns.org/)

## Disclaimer

This is a community-style example based on a personal deployment pattern, **not an official, turnkey installation**. Review current upstream documentation, adapt network/firewall/security choices to your environment, and test before relying on it for remote access.

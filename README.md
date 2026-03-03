# 🏠 Home Lab

Personal home lab running on a repurposed laptop. Started this as a way to get hands-on with self-hosting, Linux administration, and network security — and it's grown into something I actually rely on day-to-day.

Still a work in progress. The core stuff is up and running; the rest is planned.
I keep this README up to date with my progress.

---

## 🖥️ Hardware

| Device     | Repurposed  - cleaned & upgraded |
|------------|-------------------------------------------|
| CPU        | Intel Core i5-4310U @ 2.00GHz (4 threads) |
| RAM        | 16GB DDR3                                 |
| Storage    | 1TB                                       |

---

## 🐧 Host OS

**Ubuntu Server 24.04.4 LTS** — headless, runs 24/7

- Security patches applied via `unattended-upgrades` + `needrestart`
- Docker container images updated via `cron` jobs
- All containers set to `restart: unless-stopped` to survive reboots
---

## 🌐 Accessing the server

| Service     | Status | Role                                                                 |
|--------------|--------|----------------------------------------------------------------------|
| **Tailscale**| ✅ Done | Free VPN provider, that also gives TLS certificates for their domain — all services accessible only over Tailscale  |
| **Caddy**    | ✅ Done | Reverse proxy — HTTPS via Tailscale-issued TLS cert (`*.ts.net`)    |
| **UFW**      | ✅ Done | Firewall blocks all traffic except Tailscale interface (`ts0`)    |
| **OpenSSH**  | ✅ Done | Key-based auth only, password auth disabled, non-default port, restricted to Tailscale interface |
| **Fail2ban** | ✅ Done | Bans IPs on 3rd failed SSH attempt for 5 mins                    |
| **NextDNS**  | ✅ Done | Cloud DNS with blocklists — coexists with Tailscale    |

> No ports are exposed on the home router. All external access goes through Tailscale.

---

## 📦 Services

All services run in **Docker** via **Docker Compose**. docker-compose.yaml examples at the bottom.

### Currently Running

| Service          | Purpose                              | Notes                                              |
|------------------|--------------------------------------|----------------------------------------------------|
| **Nextcloud**    | Self-hosted file storage & sharing   | Friend accounts set up for sharing                 |
| **Vaultwarden**  | Self-hosted password manager         | Bitwarden-compatible clients & browser extensions  |
| **Caddy**        | Reverse proxy                        | HTTPS termination with Tailscale TLS cert          |

### Planned

| Service          | Purpose                              | Notes                                              |
|------------------|--------------------------------------|----------------------------------------------------|
| **Portainer**    | Docker management UI                 | Web UI over Tailscale — works on CLI-only servers  |
| **Wazuh**        | SIEM — log aggregation & alerts      | Agents on server, main PC, and travel laptop       |
| **Suricata**     | IDS — network traffic inspection     | Custom rules, alert validation                     |
| **Authelia**     | SSO + 2FA                            | Single login across all services                   |
| **Uptime Kuma**  | Uptime monitoring & alerting         | Alerts via email/Telegram if anything goes down    |
| **qBittorrent**  | Torrent client                       | Web UI, accessible via Tailscale only              |

---

## 📊 Monitoring


Planning to deploy:
- **Wazuh** — centralised log aggregation across the server, main PC, and travel laptop. The server will not be able to handle the log querries.
- **Uptime Kuma** — service uptime monitoring with email alerts

The focus will be on log aggregation and alerting, not active response. 
Planning on setting up rules to catch network traffic, VPN logins and ssh logs.

---

## 🔒 Security Posture

**Done:**
- ✅ No open ports on home router — Tailscale-only access
- ✅ SSH hardened — key-only auth, non-default port, restricted to Tailscale interface
- ✅ UFW configured — all traffic blocked except through `ts0`
- ✅ Secrets managed via `.env` files — never hardcoded in compose files
- ✅ DNS-level blocking via **NextDNS**
- ✅ **Fail2ban** for SSH brute-force protection

**Planned:**
- 🔜 All services behind **Authelia SSO + 2FA**
- 🔜 IDS via **Suricata** with custom rules
- 🔜 SIEM via **Wazuh** aggregating logs from all devices
- 🔜 Enroll all personal devices to gather logs from the main PC, server, and laptop


---

## 🗂️ Example Compose Configs

<details>
<summary>📄 Nextcloud — docker-compose.yml</summary>

```yaml
services:
  nextcloud:
    image: nextcloud:33
    restart: unless-stopped
    ports:
      - "8080:80"
    depends_on:
      - nextcloud-db
    environment:
      MYSQL_HOST: nextcloud-db
      MYSQL_DATABASE: nextcloud
      MYSQL_USER: nextcloud
      MYSQL_PASSWORD: ${MYSQL_PASSWORD}
      NEXTCLOUD_ADMIN_USER: admin
      NEXTCLOUD_ADMIN_PASSWORD: ${NEXTCLOUD_ADMIN_PASSWORD}
      NEXTCLOUD_TRUSTED_DOMAINS: "localhost syntac-server.ts.net"
      OVERWRITEPROTOCOL: https
      OVERWRITEHOST: syntac-server.ts.net
      TRUSTED_PROXIES: "172.16.0.0/12 192.168.0.0/16 10.0.0.0/8 100.64.0.0/10"
    volumes:
      - /home/syntac/nextcloud/nextcloud-data:/var/www/html
    networks:
      - nextcloud-net

  nextcloud-db:
    image: mariadb:lts
    restart: unless-stopped
    environment:
      MYSQL_ROOT_PASSWORD: ${MYSQL_ROOT_PASSWORD}
      MYSQL_DATABASE: nextcloud
      MYSQL_USER: nextcloud
      MYSQL_PASSWORD: ${MYSQL_PASSWORD}
    volumes:
      - nextcloud-db:/var/lib/mysql
    networks:
      - nextcloud-net

volumes:
  nextcloud-db:

networks:
  nextcloud-net:
    driver: bridge
```

</details>

---

<details>
<summary>📄 Vaultwarden — docker-compose.yaml</summary>

```yaml
services:
  vaultwarden:
    image: vaultwarden/server:latest
    container_name: vaultwarden
    restart: unless-stopped
    volumes:
      - ./data:/data
    ports:
      - "127.0.0.1:8081:80"
      - "127.0.0.1:3012:3012"
    environment:
      - WEB_VAULT_ENABLED=true
      - DOMAIN=https://syntac-server.ts.net/vault
      - SIGNUPS_ALLOWED=false
      - ADMIN_TOKEN=${ADMIN_TOKEN}
      - WEBSOCKET_ENABLED=true
      - LOG_FILE=/data/vaultwarden.log
      - LOG_LEVEL=warn
    networks:
      proxy:
        ipv4_address: 172.20.0.10

networks:
  proxy:
    external: true
```

</details>

---

<details>
<summary>📄 Caddy — docker-compose.yaml</summary>

```yaml
services:
  caddy:
    image: caddy:latest
    container_name: caddy
    restart: unless-stopped
    network_mode: host
    volumes:
      - ./Caddyfile:/etc/caddy/Caddyfile
      - ./certs/syntac-server.ts.net.crt:/certs/syntac-server.ts.net.crt
      - ./certs/syntac-server.ts.net.key:/certs/syntac-server.ts.net.key
      - caddy_data:/data
      - caddy_config:/config

volumes:
  caddy_data:
  caddy_config:
```

</details>

---

<details>
<summary>📄 Caddy — Caddyfile</summary>

```
syntac-server.ts.net {
    tls /certs/syntac-server.ts.net.crt /certs/syntac-server.ts.net.key

    handle /vault* {
        reverse_proxy 127.0.0.1:8081
    }

    handle /notifications/hub* {
        reverse_proxy 127.0.0.1:3012
    }

    handle {
        reverse_proxy localhost:8080 {
            header_up X-Real-IP {remote_host}
            header_up X-Forwarded-For {remote_host}
            header_up X-Forwarded-Proto {scheme}
        }
    }
}
```

</details>

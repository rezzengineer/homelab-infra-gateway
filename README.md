# 🏠 homelab-infra-gateway

> **A production-grade network infrastructure stack for self-hosted environments.**
> Stop exposing services directly. Start routing intelligently, encrypting automatically, and filtering at the DNS level — all within your own infrastructure.

[![Docker](https://img.shields.io/badge/Docker-Compose-2496ED?style=flat-square&logo=docker&logoColor=white)](https://docs.docker.com/compose/)
[![Nginx Proxy Manager](https://img.shields.io/badge/Nginx_Proxy_Manager-Latest-F15833?style=flat-square&logo=nginx&logoColor=white)](https://nginxproxymanager.com/)
[![AdGuard Home](https://img.shields.io/badge/AdGuard_Home-Latest-68BC71?style=flat-square&logo=adguard&logoColor=white)](https://adguard.com/en/adguard-home/overview.html)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg?style=flat-square)](LICENSE)
[![Self-Hosted](https://img.shields.io/badge/Self--Hosted-Ready-blueviolet?style=flat-square)]()

---

## 📖 Overview

This repository provisions the **foundational network gateway layer** for a self-hosted homelab or small-office infrastructure. The stack is designed around two critical concerns: **traffic management** and **network-level security**.

| Component | Role |
| :--- | :--- |
| **Nginx Proxy Manager** | Reverse proxy engine with a GUI dashboard. Handles SSL/TLS termination via Let's Encrypt, routing rules, and access control — all without touching a single Nginx config file manually. |
| **AdGuard Home** | A self-hosted DNS server that blocks ads, trackers, and malicious domains at the network level — before a single packet reaches a device. No client-side extension required. |

Together, they form a **unified ingress and DNS control plane** that is lightweight, auditable, and fully self-managed.

---

## 🗺️ Network Architecture

The diagram below illustrates how traffic flows through the stack: all inbound requests enter through NPM, which routes to internal services over a dedicated Docker network. All DNS queries from LAN devices are resolved by AdGuard Home before being forwarded upstream.

![Diagram Arsitektur](assets/architecture-diagram.png)

> 💡 *Tip: Tools like [draw.io](https://draw.io), [Excalidraw](https://excalidraw.com), or [Mermaid Live Editor](https://mermaid.live) can be used to create this diagram.*

---

## ✅ Prerequisites

Before deploying, ensure your host machine satisfies the following requirements:

- **OS:** Any modern Linux distribution (Debian 12, Ubuntu 22.04 LTS, or equivalent recommended).
- **Runtime:** Docker Engine `≥ 24.x` and Docker Compose Plugin `≥ 2.x` installed.
- **Privileges:** A non-root user with `sudo` access and membership in the `docker` group.
- **Network:** The following host ports must be **free and unoccupied** before deployment.

### 🔌 Port Reference

| Service | Host Port | Container Port | Protocol | Purpose |
| :--- | :---: | :---: | :---: | :--- |
| **NPM** | `80` | `80` | TCP | Standard HTTP ingress (reverse proxy). |
| **NPM** | `443` | `443` | TCP | Standard HTTPS/SSL ingress (reverse proxy). |
| **NPM UI** | `81` | `81` | TCP | Admin dashboard for Nginx Proxy Manager. |
| **AdGuard DNS** | `53` | `53` | TCP/UDP | Standard DNS resolution port. **Must be free.** |
| **AdGuard Setup** | `3000` | `3000` | TCP | Initial setup wizard (first-run only). |
| **AdGuard UI** | `8080` | `80` | TCP | Admin dashboard after setup is complete. |

> ⚠️ **Critical — Port 53 Conflict on Debian/Ubuntu:**
> Modern Ubuntu and Debian systems run `systemd-resolved` which binds to port `53` by default. This **will** cause a conflict. Resolve it before deploying:
>
> ```bash
> # Disable the systemd-resolved stub listener
> sudo sed -i 's/#DNSStubListener=yes/DNSStubListener=no/' /etc/systemd/resolved.conf
> sudo systemctl restart systemd-resolved
>
> # Verify port 53 is now free
> sudo ss -tulpn | grep ':53'
> ```

---

## 🚀 Deployment

### 1. Clone the Repository

```bash
git clone https://github.com/rezzengineer/homelab-infra-gateway.git
cd homelab-infra-gateway
```
### 2. Launch the Stack

```bash
docker compose up -d
```

Verify all containers are running and healthy:

```bash
docker compose ps
docker compose logs -f
```

---

## 🛠️ Post-Installation Setup

Once the stack is running, each service requires a one-time initialization via its web UI.

---

### 1 · AdGuard Home — Initial Setup Wizard

AdGuard Home's setup wizard is only available on port `3000` during first-run. After completion, the admin dashboard permanently moves to port `8080`.

**Navigate to the setup wizard:**

```
http://<HOST-IP>:3000
```

![AdGuard Setup — Welcome Screen](assets/adguard-setup-1.png)

**Step through the wizard:**

1. **Getting Started** — Review the welcome screen and click **Get Started**.

2. **Admin Web Interface** — When asked to set the Web Interface port, **leave it at `80`**. Docker Compose already maps container port `80` → host port `8080`, so this is correct.

   ![AdGuard Setup — Interface Port Configuration](assets/adguard-setup-2.png)

3. **DNS Server** — Leave the DNS server listen address as `All Interfaces:53`. This allows all LAN devices to use AdGuard as their resolver.

   ![AdGuard Setup — DNS Configuration](assets/adguard-setup-3.png)

4. **Authentication** — Create a strong **username** and **password** for the admin account.

   ![AdGuard Setup — Admin Credentials](assets/adguard-setup-4.png)

5. **Configure Your Devices** — AdGuard Home will display the DNS addresses it is currently listening on, along with per-platform setup instructions (Router, Windows, macOS, Android, iOS, and DNS Privacy). For the broadest coverage with zero per-device effort, follow the **Router** tab: point your router's DHCP DNS field to the host IP, and every device on the network will automatically use AdGuard as its resolver.

   ![AdGuard Setup — Configure Your Devices](assets/adguard-setup-5.png)

   > 💡 *The addresses shown (e.g. `127.0.0.1`, `172.17.0.3`) are the internal Docker network addresses AdGuard is bound to. For LAN-wide DNS filtering, use your **host machine's LAN IP** (e.g. `192.168.1.x`) when configuring your router's DNS settings.*

   Click **Next** to proceed.

6. **Finish** — The wizard will display a **"Congratulations!"** confirmation screen indicating that the setup procedure is complete. Click **Open Dashboard** to be redirected to the main admin interface.

   ![AdGuard Setup — Setup Complete](assets/adguard-setup-6.png)

After setup, the admin dashboard is permanently accessible at:

```
http://<HOST-IP>:8080
```

7. **Admin Dashboard** — You are now inside the AdGuard Home control panel. From here you can monitor DNS query statistics in real time, manage blocklists, review the query log, and configure upstream DNS resolvers.

   ![AdGuard Home — Admin Dashboard](assets/adguard-dashboard.png)
---

### 2 · Nginx Proxy Manager — First-Time Setup

Access the NPM admin dashboard:

```
http://<HOST-IP>:81
```

On first run, NPM does **not** present a login page. Instead, it displays a **"Welcome!"** setup wizard prompting you to create your admin account directly.

![NPM — First-Time Setup Wizard](assets/npm-setup-wizard.png)

Fill in the following fields and click **Save**:

| Field | Notes |
| :--- | :--- |
| **Full Name** | Your display name (e.g. `Admin`). |
| **Email Address** | This will be your login username going forward. Use a real address. |
| **New Password** | Choose a strong password. This replaces the old default `changeme`. |

> ⚠️ **There are no default credentials to "just try first."** NPM requires you to define your own admin account on first access. Store these credentials securely.

After clicking **Save**, you will be redirected to the main NPM admin dashboard. You are now ready to create your first **Proxy Host** and issue SSL certificates via Let's Encrypt.

![NPM — Admin Dashboard](assets/npm-dashboard.png)
---

## 🔒 Security & Hardening Notes

Deploying infrastructure without hardening it is an operational liability. The following measures are **strongly recommended** as immediate post-deployment steps.

### Rotate Default Credentials Immediately

NPM ships with well-known default credentials (`admin@example.com` / `changeme`). These must be changed upon first login. Do not operate the stack with default credentials even in a trusted LAN environment.

### Restrict Management UI Access to LAN / VPN Only

Ports `81` (NPM UI) and `8080` (AdGuard UI) are **management interfaces** and should never be reachable from the public internet. Enforce this at the firewall level:

```bash
# Example using UFW: Allow UI ports only from your local subnet
sudo ufw allow from 192.168.1.0/24 to any port 81
sudo ufw allow from 192.168.1.0/24 to any port 8080

# Deny from everywhere else
sudo ufw deny 81
sudo ufw deny 8080
```

If your homelab is accessed remotely, place these ports behind a **VPN** (e.g., WireGuard) rather than exposing them directly.

### AdGuard DNS — Point LAN Devices to the Host IP

For AdGuard to filter your network, configure your router's DHCP server to advertise your host machine's IP as the **primary DNS server**. This ensures all connected devices resolve DNS through AdGuard Home without any per-device configuration.

### Enable HTTPS for All Proxied Services

Use NPM to issue Let's Encrypt certificates for every internal service proxied through it. Avoid operating services over plain HTTP, even on a LAN, to prevent credential interception on shared networks.

### Review AdGuard Blocklists Regularly

The default blocklists are a good starting point, but tuning them to your environment prevents false positives. Review the **Query Log** in the AdGuard dashboard periodically and whitelist legitimate domains that may be incorrectly blocked.

---

## 📁 Repository Structure

```
homelab-infra-gateway/
├── docker-compose.yml      # Main stack definition
├── .env.example            # Environment variable template
├── assets/                 # Screenshots and architecture diagrams
│   ├── architecture-diagram.png
│   ├── adguard-setup-1.png
│   └── npm-dashboard.png
└── README.md
```

---

## 🤝 Contributing & Feedback

This repository is part of a personal infrastructure portfolio. Issues, suggestions, and pull requests are welcome. If you are adapting this stack for your own homelab or office environment, feel free to open a discussion.

---

<p align="center">
  Built with deliberate intent. Designed for reliability. Maintained with care.
  <br/>
  <sub>© 2025 · Self-Hosted Infrastructure Portfolio</sub>
</p>

# PROXMOX HOMELAB

## Hardware:
- Mini PC (256GB SSD, 16GB RAM)
- Unifi Flex Mini 2.5 Switch
- Ugreen USB C to Ethernet Adapter 2.5G
- Ugreen USB to USB C adapter 10Gbps & 3A
- 5-Pack Cable Matters 1ft 10Gbps Cat6 Ethernet Patch Cables

## Hardware Setup:
- Cat6 Cable from the wall plugged into nic0 of optiplex (vmbr0)
- optiplex USB port > USB/USBC adapter > USB C to Ethernet Adapter > Cat6 Patch Cable > Port 2 of flex mini switch (vmbr1)


## Proxmox And Network Setup / Topology:
- My home network IP gateway is 192.168.X.X, so all of my home network DHCP IPs are gonna be 192.168.X.xxx
- I have OPNsense virtualized in Proxmox as a VM, which configures vmbr0 as WAN (192.168.X.XX/24) and vmbr1 as LAN (10.0.0.1/24)
- I changed my Proxmox gateway and management IP to 10.0.0.XX, with the gateway of 10.0.0.1 so it's only accessed through things on my Proxmox LAN
- I set up the OPNsense Tailscale plugin with OPNsense as an exit node that advertises all VLAN subnets for secure remote access

## VLAN Layout:

| VLAN       | Tag | Subnet         | DHCP Range                  | Purpose                                                      |
| ---------- | --- | -------------- | --------------------------- | ------------------------------------------------------------ |
| LAN        | —   | 10.0.0.0/24    | 10.0.0.50 – 10.0.0.200     | Proxmox host, trusted wired devices                          |
| mgmt       | 10  | 10.0.10.0/24   | 10.0.10.50 – 10.0.10.200   | Management plane — Proxmox, OPNsense, Unifi, AdGuard, Homarr |
| services   | 20  | 10.0.20.0/24   | 10.0.20.50 – 10.0.20.200   | Self-hosted services — `homelab-site` LXC, Jellyfin (planned) |
| automation | 30  | 10.0.30.0/24   | 10.0.30.50 – 10.0.30.200   | `agent60` VM, agent60, automation                           |
| iot        | 40  | 10.0.40.0/24   | 10.0.40.50 – 10.0.40.200   | IoT / smart home devices (future)                            |

All DHCP servers configured to advertise AdGuard Home (10.0.10.XX) as DNS.

## Firewall Rules:

### LAN

| Action | Protocol | Source      | Destination        | Description                  |
| ------ | -------- | ----------- | ------------------ | ---------------------------- |
| Block  | IPv4 *   | LAN network | mgmt network       | Block LAN → mgmt             |
| Block  | IPv4 *   | LAN network | services network   | Block LAN → services         |
| Block  | IPv4 *   | LAN network | automation network | Block LAN → automation       |
| Pass   | IPv4 *   | LAN network | *                  | Allow LAN to internet        |

> Note: LAN devices access DNS via OPNsense (10.0.0.1), which forwards to AdGuard internally.
> Mgmt UIs (Homarr, Proxmox, OPNsense) are accessed via Tailscale when on LAN or remotely.

### mgmt

| Action | Protocol | Source       | Destination | Port | Description                                                                         |
| ------ | -------- | ------------ | ----------- | ---- | ----------------------------------------------------------------------------------- |
| Pass   | TCP/UDP  | 10.0.10.XX  | *           | 53   | AdGuard bootstrap DNS — must be first; resolves DoH upstream (dns10.quad9.net) before RFC1918 block |
| Pass   | TCP/UDP  | mgmt network | 10.0.10.XX | 53   | DNS to AdGuard (mgmt devices)                                                       |
| Block  | *        | mgmt network | RFC1918     | *    | Block all internal nets                                                             |
| Pass   | TCP      | mgmt network | *           | 443  | HTTPS — updates, Ubiquiti cloud, Proxmox community repo, AdGuard DoH upstream       |
| Pass   | TCP      | mgmt network | *           | 80   | HTTP — apt package repos                                                            |
| Block  | *        | mgmt network | *           | *    | Default deny                                                                        |

### services

| Action | Protocol | Source           | Destination | Port | Description                                                       |
| ------ | -------- | ---------------- | ----------- | ---- | ----------------------------------------------------------------- |
| Pass   | TCP/UDP  | services network | 10.0.10.XX | 53   | DNS to AdGuard                                                    |
| Block  | *        | services network | RFC1918     | *    | Block lateral access to other internal nets                       |
| Pass   | TCP      | services network | *           | 443  | HTTPS — updates, Jellyfin metadata (TMDB / TVDB), GitHub SSH-via-443 |
| Pass   | TCP      | services network | *           | 80   | HTTP — apt package repos                                          |
| Block  | *        | services network | *           | *    | Default deny                                                      |

> Note: LAN → services stays blocked. Jellyfin (and future services) reached via Tailscale only — the OPNsense Tailscale plugin advertises 10.0.20.0/24 to the tailnet, so tailnet devices have L3 reach without an extra pass rule.

### automation

| Action | Protocol | Source             | Destination | Port | Description                                   |
| ------ | -------- | ------------------ | ----------- | ---- | --------------------------------------------- |
| Pass   | TCP/UDP  | automation network | 10.0.10.XX | 53   | DNS to AdGuard                                |
| Block  | *        | automation network | RFC1918     | *    | No access to any internal network             |
| Pass   | TCP      | automation network | *           | 443  | HTTPS — Anthropic, Notion, Google, Open-Meteo |
| Pass   | TCP      | automation network | *           | 80   | HTTP — apt package repos                      |
| Pass   | TCP      | automation network | *           | 465  | SMTP SSL — SMTP provider                          |
| Block  | *        | automation network | *           | *    | Deny everything else                          |

### iot
Rules not yet configured — no devices on this VLAN.

### WAN / Tailscale
No inbound rules (default deny). Tailscale subnet routing handled by OPNsense Tailscale plugin.


## LXC & VM Resource Allocations with IPs:
- LXCs: Unifi OS Server, Homarr, AdGuard Home, homelab-site
- VMs: OPNsense, agent60 VM (in progress)

- Unifi OS Server (10.0.10.XX): 4GiB memory, 512MiB swap, 2 cores — mgmt VLAN
- Homarr (10.0.10.XX): 2GiB memory, 512MiB swap, 2 cores — mgmt VLAN
- AdGuard Home (10.0.10.XX): 2GiB memory, 2GiB swap, 1 core — mgmt VLAN
- homelab-site (10.0.20.XX, DHCP reserved): 256MiB memory, 256MiB swap, 1 core, 4GiB disk — services VLAN (tag 20)
- agent60 VM (10.0.30.x, DHCP reserved): 2GiB memory, 512MiB swap, 2 cores — automation VLAN (tag 30)
- OPNsense (WAN: 192.168.X.XX, LAN: 10.0.0.1, mgmt: 10.0.10.1): 2GiB/4GiB, 1 socket, 4 cores

## LXC & VM Functions:

### OPNsense (VM)
- **Role:** Does all the actual networking work — this is where the VLANs *live*.
- **What it actually does:**
  - **Creates and enables each VLAN** (10 / 20 / 30 / 40) on the `vmbr1` parent interface inside Interfaces → Other Types → VLAN
  - **Configures the VLAN interfaces** themselves (assigns subnet, gateway IP, MTU, etc.) and enables each one under Interfaces → Assignments
  - Runs **DHCP** on every VLAN (advertises AdGuard 10.0.10.XX as DNS)
  - Acts as the **default gateway** for every subnet (10.0.0.1 / 10.0.10.1 / 10.0.20.1 / 10.0.30.1 / 10.0.40.1)
  - Enforces all **firewall rules** between VLANs and to/from WAN
  - Forwards LAN DNS queries to AdGuard; handles **NAT** for WAN egress
- **Plugins:** Tailscale (exit node, advertises all VLAN subnets for remote access)
- **Interfaces:** WAN (vmbr0), LAN + tagged VLANs (vmbr1)

### Unifi OS Server (LXC — mgmt VLAN)
- **Role:** Tells the switch which VLAN tags are allowed to pass — that's it. OPNsense does all the real networking; this controller is just the dumb-pipe configurator for the Flex Mini.
- **What it actually does:**
  - **Defines the VLAN tag numbers** (10 / 20 / 30 / 40) in the controller so the switch knows what tags to forward (it doesn't define the subnets, gateways, or DHCP — that's OPNsense's job)
  - Configures **Port 2** as the OPNsense uplink (Mini PC USB → USB-C NIC → vmbr1):
    - **Native (untagged) VLAN = LAN 10.0.0.0/24** — anything sent untagged on Port 2 lands on LAN
    - **Trunks tagged VLANs 10 / 20 / 30 / 40** on top so the switch carries every VLAN's tagged frames to/from OPNsense
  - Remaining ports get assigned an access VLAN per attached device as they get plugged in
- **Division of labor with OPNsense:** OPNsense creates and enables the VLAN interfaces, runs DHCP, acts as the gateway, and enforces firewall rules — *all* the actual work. The switch (configured by Unifi OS) just needs the matching tag list so it doesn't drop the frames in transit.
- **Network:** Switch sits on LAN physically but is overridden to mgmt VLAN (10.0.10.x) via Unifi OS

### AdGuard Home (LXC — mgmt VLAN)
- **Role:** Network-wide DNS resolver, ad/tracker blocker, and local DNS for `.homelab` URLs
- **Upstream DNS:** https://dns10.quad9.net/dns-query (Quad9 DoH — malware-blocking endpoint, served over HTTPS:443)
- **Used by:** Every LXC + VM (and any Tailscale client) — every DHCP server on OPNsense advertises 10.0.10.XX as the DNS server, and the OPNsense Tailscale plugin pushes AdGuard as the DNS for tailnet devices too
- **DNS rewrites (local names):** AdGuard rewrites a set of `.homelab` hostnames to the right internal IPs so I can hit memorable URLs from any device on LAN/VLAN/Tailscale instead of memorizing IPs. Examples:
  - `proxmox.homelab` → 10.0.0.XX
  - `opnsense.homelab` → 10.0.10.1
  - `adguard.homelab` → 10.0.10.XX
  - `unifi.homelab` → 10.0.10.XX
  - `homarr.homelab` → 10.0.10.XX
  - `agent.homelab` → 10.0.30.XX

### Homarr (LXC — mgmt VLAN)
- **Role:** Homelab dashboard — service links and status overview
- **Access:** Via Tailscale only (LAN → mgmt is blocked at firewall)

### homelab-site (LXC — services VLAN)
- **Role:** Static nginx server hosting this very document's HTML version (`homelab_reorganization.html`) for personal viewing — basically a self-hosted, always-current copy of my homelab docs.
- **Stack:** Debian 12 · nginx serving `/opt/homelab-site/` as the document root with `homelab_reorganization.html` set as the directory index — so the bare URL renders the page with no port and no path suffix. Reachable at `http://map.homelab` (AdGuard rewrite → 10.0.20.XX).
- **Static assets in the document root (served alongside the HTML by the same nginx config):**
  - `favicon.png` — custom-drawn icon referenced by `<link rel="icon" type="image/png" href="favicon.png">` in the HTML. This is what modern browsers display in the tab.
  - `favicon.ico` — legacy fallback. Browsers always auto-issue `GET /favicon.ico` regardless of the `<link>` tag; keeping a file here means those requests succeed (no `404` lines spamming the nginx access log) and very old clients still get an icon. The `<link>`-referenced `.png` still wins as the actual displayed tab icon.
- **Access:** Tailscale-only. LAN → services is blocked at the firewall; OPNsense's Tailscale plugin advertises 10.0.20.0/24 to the tailnet, so any tailnet device has L3 reach without a port-forward or per-port pass rule. No public exposure of any kind.
- **Repo + deploy:**
  - Private GitHub repo cloned via SSH using a **read-only deploy key** (repo-scoped, not user-scoped) generated on the LXC and added under repo Settings → Deploy keys with the "Allow write access" checkbox unchecked.
  - SSH is tunneled through HTTPS:443 (`Host github.com → HostName ssh.github.com Port 443` in `~/.ssh/config`) because the services VLAN only allows :443 outbound, not :22 — same trick the automation VM uses for the agent60 repo.
  - Manual `git pull` to update — no webhook, no public listener, no cron. Static content is auto-served by nginx on next request.
- **Blast radius if compromised:** read-only access to one repo, nothing else. No write access to push back to GitHub, no other repos, no Anthropic / SMTP / agent60 credentials (those live on the isolated automation VM).
- **Role:** Custom automation VM — isolated on VLAN 30 with internet-only access
- **OS:** Ubuntu (default user account)
- **Repo:** `/opt/agent60` — cloned from `git@github.com:<user>/agent60.git`

#### agent60 (set up 2026-05-10)
- Sends AI-generated email reports using Claude, Google Calendar, and Open-Meteo
- **Monday Brief** — upcoming week personal events + 7-day weather forecast (cron: `0 10 * * 1` UTC = 5:00 AM local)
- **Daily Brief** — today's personal calendar events + weather prep tip (cron: `0 10 * * 2-7` UTC = 5:00 AM local, Tue–Sun)
- **Friday Report** — weekly wrap-up of online coding class notes transcribed in Notion + calendar summary (cron: `0 23 * * 5` UTC = 6:00 PM local)
- VM runs in UTC — cron times offset to match local time (UTC offset)
- Email sent via SMTP provider (msmtp) — `smtp.example.com:465` (implicit TLS)
- **AdGuard bootstrap DNS rule** — adding "deny everything else" to mgmt broke DNS network-wide by blocking AdGuard's port 53 bootstrap queries to resolve its DoH upstream (dns10.quad9.net). Fixed 2026-05-16: added `Pass TCP/UDP 10.0.10.XX → *:53` as the first mgmt rule, before the RFC1918 block
- **AppArmor blocking msmtp** — AppArmor restricts msmtp from reading config files outside the home directory. Fixed 2026-05-12: copied `.msmtprc` to `~/.msmtprc` and removed `--file=` flag from all scripts so msmtp reads it automatically
- **Cron PATH missing claude** — cron runs with a stripped PATH; `claude` at `/home/user/.local/bin/claude` not found. Fixed 2026-05-12: added `PATH=/home/user/.local/bin:/usr/local/bin:/usr/bin:/bin` as the first line of crontab
- **IPv6 disabled** — automation VLAN is IPv4-only; msmtp resolved to IPv6 first causing connection failures. Fixed 2026-05-11:
  ```bash
  echo "net.ipv6.conf.all.disable_ipv6 = 1
  net.ipv6.conf.default.disable_ipv6 = 1" | sudo tee /etc/sysctl.d/99-no-ipv6.conf
  sudo sysctl --system
  ```

#### GitHub SSH Setup (set up 2026-05-10)
- SSH key pair: `~/.ssh/id_ed25519` (ed25519, no passphrase) — added to a GitHub account
- SSH config routes through `ssh.github.com:443` — automation VLAN only allows port 443 outbound, not 22
- `keychain` installed — SSH agent auto-starts on login, silent `git pull` with no passphrase prompts

#### GitHub Webhook + ngrok Auto-Deploy (set up 2026-05-10)
Automatically pulls the latest code to `/opt/agent60` on every push to `main`.

**Services (both systemd, enabled, auto-start on boot):**
- `webhook.service` — Node.js HTTP server at `localhost:3000` (`/opt/webhook/index.js`)
  - Receives GitHub push events, verifies HMAC-SHA256 signature, runs `git -C /opt/agent60 pull`
  - 5s timeout on `execSync` to prevent hanging
  - Logs to journald: `journalctl -u webhook --no-pager`
- `ngrok.service` — exposes `localhost:3000` via a static ngrok-free.app domain over HTTPS
  - Config: `/home/user/.config/ngrok/ngrok.yml`

**GitHub webhook settings (repo → Settings → Webhooks):**
- Payload URL: `https://<static-domain>.ngrok-free.app/webhook`
- Content type: `application/json`
- Events: push only
- Secret: HMAC secret matching `SECRET` in `index.js`

**Useful commands:**
```bash
journalctl -fu webhook                                          # live logs
journalctl -u webhook --no-pager | grep -E "Pull|Rejected|Webhook"  # filtered history
systemctl status webhook ngrok                                  # service health
```

## Security Hardening:

### Completed
- **2FA on Proxmox** — TOTP enabled on user account (Datacenter → Users → Edit) — 2026-05-11
- **2FA on OPNsense** — TOTP enabled on admin account (System → Access → Users) — 2026-05-11
- **Secret file permissions tightened on `agent60 VM`** — 2026-05-11:
  ```
  /opt/agent60/.env                 chmod 600  (owner read/write only)
  /opt/agent60/.msmtprc             chmod 600  (owner read/write only)
  /home/user/.config/ngrok/ngrok.yml  chmod 600  (owner read/write only)
  /opt/webhook/index.js                  chmod 640  (owner read/write, group read)
  ```
- **Webhook `execSync` timeout** — 5s limit prevents hanging git pull from blocking the listener
- **Webhook rate limiting** — per-IP 5s cooldown; returns 429 Too Many Requests on repeat hits (`/opt/webhook/index.js`)
- **agent60 LXC → VM conversion complete** — 2026-05-12: migrated to Ubuntu VM for stronger kernel-level isolation; ngrok + webhook services carried over
- **SMTP provider** — dedicated SMTP provider account used solely for agent60 sending; isolated from personal email
- **VLAN segmentation** — all containers isolated by function; automation VLAN internet-only, no internal access
- **Proxmox management off WAN** — host only reachable via internal LAN (10.0.0.XX)
- **Default deny on WAN + Tailscale** — no inbound rules on either interface
- **IPv6 disabled on `agent60 VM`** — prevents msmtp routing issues on IPv4-only VLAN
- **Strong randomized password for `agent60` VM** — 2026-05-13: replaced the default/setup password on the default user account with a long random string from a password manager's password generator (stored in the same password-manager vault). Makes password-based SSH safe to leave enabled as a fallback alongside the ed25519 key.
- **Read-only deploy key for `homelab-site` repo clone** — 2026-05-17: generated a dedicated ed25519 key on the `homelab-site` LXC and added it to the GitHub repo under Settings → Deploy keys with **"Allow write access" unchecked**. Repo-scoped (not tied to my user account), pull-only, and easy to revoke independently of any other token. If the LXC is ever compromised the attacker only gets read access to this one repo — no push, no other repos, no account-wide blast radius.
- **Proxmox host firewall enabled** — 2026-05-17: turned on at Datacenter → Firewall as a second layer beyond OPNsense, so the Proxmox host itself filters traffic regardless of what's happening inside the guest network.

### In Progress

### Pending
- Tailscale ACLs — restrict which devices can reach which subnets
- Move Homarr to services VLAN (currently on mgmt)

## My Homelab Goals:
- Self-host services securely so in case of LXCs or VMs getting compromised, they stay contained and don't infect anything
- Securely remote access my homelab
- Learn Proxmox, Docker, Linux, networking, etc.

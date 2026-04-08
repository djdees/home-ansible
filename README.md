# home-ansible

Single-repository Ansible configuration for the home network.

---

## Network

| Host | IP | Role |
|---|---|---|
| `muspell` | `192.168.1.198` | Control machine (Ubuntu 24.04) |
| `garmr` | `192.168.1.5` | Gateway |
| `pihole` | `192.168.1.8` | Pi-hole v6 + Unbound (DNS) |
| `scribe` | `192.168.1.83` | Raspberry Pi -- Nginx web server |
| `hunin` | `192.168.1.84` | Ubuntu 24.04 -- Docker host (ClickHouse, Grafana, proxy) |
| `munin` | `192.168.1.85` | Ubuntu 24.04 -- Docker host |

Domain: `home.arpa`

---

## Repository layout

```
home-ansible/
├── ansible.cfg
├── .yamllint
│
├── inventories/
│   └── home/
│       ├── hosts.yml                    <- all hosts (one source of truth)
│       ├── host_vars/
│       │   ├── hunin/vars.yml           <- ClickHouse paths, OTEL, Grafana, proxy
│       │   ├── munin/vars.yml           <- static IP
│       │   ├── muspell/vars.yml         <- OTEL settings (control machine)
│       │   ├── pihole/vars.yml          <- conservative OTEL settings
│       │   └── scribe/vars.yml          <- Nginx site definitions, OTEL settings
│       └── group_vars/
│           ├── all/vars.yml             <- ansible_user, timezone, gateway, domain, NTP
│           ├── all/vault.yml            <- symlink -> ~/ansible/vault/all.yml
│           ├── pi_nodes/vars.yml        <- explicit ansible_user reminder for Pis
│           ├── pi_servers/vars.yml      <- ansible_user for scribe etc.
│           ├── infra_servers/vars.yml   <- comment only (fixer inherited from all)
│           ├── piholes/vars.yml         <- pihole vars
│           └── piholes/vault.yml        <- symlink -> ~/ansible/vault/piholes.yml
│
├── roles/
│   ├── base_packages/     <- apt cache, base packages -- dependency of most roles
│   ├── base_linux/        <- dist-upgrade; depends on base_packages + base_network
│   ├── base_pi/           <- Pi locale, timezone, network, NTP
│   ├── base_network/      <- static IP -- auto-detects NM/netplan/dhcpcd/networkd
│   ├── base_user/         <- creates djdees with passwordless sudo and SSH key
│   ├── base_ssh/          <- hardens sshd_config
│   ├── user_dotfiles/     <- bashrc, vimrc, shell selection
│   ├── base_dev/          <- Go, tmux, jq, yamllint, shellcheck
│   ├── base_docker/       <- Docker CE -- Ubuntu + Debian repo detection
│   ├── base_cups/         <- CUPS + HP drivers + Avahi AirPrint
│   ├── base_samba/        <- Samba file server
│   ├── clickhouse/        <- ClickHouse in Docker -- attached storage, OTEL schema
│   ├── grafana/           <- Grafana OSS in Docker -- ClickHouse datasource, dashboards
│   ├── proxy/             <- Chainguard nginx reverse proxy -- TLS termination, HTTP->HTTPS
│   ├── otel_collector/    <- OpenTelemetry Collector (contrib) -> ClickHouse
│   ├── pdf_server/        <- Apache2 PDF/media server
│   ├── telegraf/          <- Telegraf metrics agent
│   ├── unbound_dns/       <- Unbound recursive resolver + internal zone
│   ├── pihole_dns/        <- Pi-hole v6 (depends on unbound_dns)
│   ├── web_nginx/         <- Nginx virtual hosts from host_vars
│   └── web_apache/        <- Apache2 virtual hosts from host_vars
│
└── playbooks/
    ├── site.yml                <- full bootstrap (pi_nodes, pi_servers, infra_servers)
    ├── base-pi-setup.yml
    ├── base-linux-setup.yml
    ├── base-ssh.yml
    ├── base-dev.yml
    ├── base-docker.yml         <- targets infra_servers
    ├── clickhouse-setup.yml    <- targets hunin
    ├── grafana-setup.yml       <- targets hunin
    ├── proxy-setup.yml         <- targets hunin
    ├── otel-setup.yml          <- targets otel_nodes (muspell, hunin, scribe, pihole)
    ├── scribe-setup.yml        <- targets pi_servers
    ├── nginx-setup.yml
    ├── apache-setup.yml
    ├── unbound-setup.yml
    ├── pihole-setup.yml
    ├── dns-stack-setup.yml
    ├── cups-setup.yml
    ├── samba-setup.yml
    ├── pdf-server-setup.yml
    ├── telegraf-setup.yml
    ├── system-info.yml
    └── localhost-info.yml
```

---

## First-time setup

### 1. Clone and configure vault

```bash
git clone <your-repo-url> ~/ansible/home-ansible
cd ~/ansible/home-ansible

mkdir -p ~/ansible/vault && chmod 700 ~/ansible/vault

# Create vault files
ansible-vault create ~/ansible/vault/all.yml
# Required keys -- see "Secrets" section below

ansible-vault create ~/ansible/vault/piholes.yml
# Required keys:
#   vault_pihole_webpassword: "YourPiholeAdminPassword"

# Symlink vault files into group_vars
ln -s ~/ansible/vault/all.yml     inventories/home/group_vars/all/vault.yml
ln -s ~/ansible/vault/piholes.yml inventories/home/group_vars/piholes/vault.yml

# Store vault password (matches ansible.cfg vault_password_file)
echo 'your-vault-password' > ~/.ansible_vault_pass && chmod 600 ~/.ansible_vault_pass
```

### 2. Install required collections

```bash
ansible-galaxy collection install ansible.posix community.general community.docker
```

### 3. Bootstrap a fresh Pi

```bash
# On the Pi: enable SSH and fixer sudoers
sudo systemctl enable --now ssh
echo 'fixer ALL=(ALL) NOPASSWD: ALL' | sudo tee /etc/sudoers.d/010_fixer-nopasswd
ssh-copy-id -i ~/.ssh/id_ed25519.pub fixer@192.168.1.XXX

# First run (Pi on DHCP)
ansible-playbook playbooks/base-pi-setup.yml --limit pihole \
  -e ansible_host=192.168.1.XXX

# After reboot at static IP
ansible-playbook playbooks/dns-stack-setup.yml
```

### 4. Bootstrap hunin (Docker + ClickHouse + OTEL + Grafana + proxy)

```bash
ansible-playbook playbooks/base-linux-setup.yml --limit hunin
ansible-playbook playbooks/base-docker.yml      --limit hunin
ansible-playbook playbooks/clickhouse-setup.yml
ansible-playbook playbooks/otel-setup.yml

# Place TLS certs before running proxy (see "Reverse proxy" section)
ansible-playbook playbooks/grafana-setup.yml
ansible-playbook playbooks/proxy-setup.yml
```

### 5. Bootstrap scribe

```bash
# First run on DHCP
ansible-playbook playbooks/scribe-setup.yml -e ansible_host=192.168.1.XXX
# Thereafter
ansible-playbook playbooks/scribe-setup.yml
```

---

## User model

| Account | Purpose | Created by |
|---|---|---|
| `fixer` | Ansible/admin -- all playbook runs | Manual at imaging time |
| `djdees` | Personal interactive sessions | `base_user` role |

`ansible_user` is set to `fixer` in `group_vars/all/vars.yml` and applies to
every host. Both accounts are permanent -- `fixer` is never removed.

---

## Observability stack

```
muspell  --+
scribe   --+
pihole   --+--> otelcol-contrib --> ClickHouse (hunin:9000) --> otel database
hunin    --+                                                         |
                                                               Grafana (hunin)
                                                          https://grafana.home.arpa
```

- **ClickHouse** (`clickhouse` role) -- Docker container on hunin, data on `/data/clickhouse`.
  Tables: `otel_traces` (30d TTL), `otel_metrics` (180d on hunin), `otel_logs` (30d TTL).
- **otelcol-contrib** (`otel_collector` role) -- host metrics, journald logs, and Docker
  stats (where enabled). Deployed to all hosts in `otel_nodes`.
- **Grafana** (`grafana` role) -- Docker container on hunin, provisioned ClickHouse
  datasource and home metrics dashboard.
- To add a host to otel: add it to `otel_nodes` in `hosts.yml`, set `otel_service_name`
  in its `host_vars`, run `otel-setup.yml --limit <host>`.

---

## Reverse proxy

Chainguard nginx (`cgr.dev/chainguard/nginx`) on hunin. Terminates TLS, redirects
HTTP to HTTPS, and proxies internal services via the `proxy_net` Docker network.

```
Internet/LAN --> proxy (hunin:80/443)
                  |
                  +--> grafana:3000   (grafana.home.arpa)
                  +--> superset:8088  (superset.home.arpa)  [future]
                  +--> ...
```

**Adding a new service:**
1. Add a conf.d template to `roles/proxy/templates/conf.d/`
2. Add a deploy task to `roles/proxy/tasks/configure.yml`
3. Connect the container to `proxy_net` in `roles/proxy/tasks/network.yml`
4. Run: `ansible-playbook playbooks/proxy-setup.yml --tags configure,network`

**TLS certificates** are expected at `/data/proxy/certs/` on hunin:
- `fullchain.pem` -- certificate + intermediates
- `privkey.pem` -- private key

For a self-signed cert (home lab):
```bash
sudo openssl req -x509 -nodes -newkey rsa:4096 \
  -keyout /data/proxy/certs/privkey.pem \
  -out /data/proxy/certs/fullchain.pem \
  -days 3650 -subj "/CN=*.home.arpa" \
  -addext "subjectAltName=DNS:*.home.arpa,DNS:home.arpa"
sudo chmod 600 /data/proxy/certs/*.pem
```

When Vault-issued certs are ready, drop them in and `docker restart proxy` -- no
Ansible changes required.

---

## DNS stack

```
LAN clients --> Pi-hole (192.168.1.8:53) --> Unbound (127.0.0.1:5335) --> Root servers
                                          \-> home.arpa zone (A + PTR records)
```

Internal DNS records are managed in `roles/unbound_dns/templates/pi-hole.conf.j2`.
To add a record, edit that file and run:

```bash
ansible-playbook playbooks/unbound-setup.yml --tags internal-dns
```

Verify:
```bash
dig @127.0.0.1 -p 5335 google.com      # from pihole
dig @192.168.1.8 grafana.home.arpa     # from any host
dig @192.168.1.8 -x 192.168.1.84      # reverse lookup
```

---

## Secrets

```bash
ansible-vault view  ~/ansible/vault/all.yml
ansible-vault edit  ~/ansible/vault/all.yml
ansible-vault rekey ~/ansible/vault/all.yml ~/ansible/vault/piholes.yml
```

### Required vault keys

| Key | File | Used by |
|---|---|---|
| `vault_wifi_ssid` | `all.yml` | `base_pi` |
| `vault_wifi_psk` | `all.yml` | `base_pi` |
| `vault_clickhouse_admin_password` | `all.yml` | `clickhouse` |
| `vault_clickhouse_otel_password` | `all.yml` | `clickhouse`, `otel_collector`, `grafana` |
| `vault_grafana_admin_password` | `all.yml` | `grafana` |
| `vault_pihole_webpassword` | `piholes.yml` | `pihole_dns` |

---

## Debugging

### Exposing secrets in task output

All sensitive tasks use `no_log: "{{ ansible_no_log_tasks | default(true) }}"`.
By default secrets are hidden. To expose output for debugging, pass the variable
at runtime -- no file edits or zip/unzip cycles needed:

```bash
ansible-playbook playbooks/grafana-setup.yml -e "ansible_no_log_tasks=false"
```

This works for any playbook. Never commit `ansible_no_log_tasks=false` to
inventory or vars files.

### Other useful debug flags

```bash
# Show full task output and SSH commands
ansible-playbook playbooks/clickhouse-setup.yml -vvv

# Dry run -- show what would change without making changes
ansible-playbook playbooks/otel-setup.yml --check

# Step through tasks one at a time
ansible-playbook playbooks/proxy-setup.yml --step

# Run only tasks matching a tag
ansible-playbook playbooks/clickhouse-setup.yml --tags schema

# Limit to specific hosts
ansible-playbook playbooks/otel-setup.yml -l hunin,scribe
```

---

## Role dependency graph

```
pihole_dns
  └── unbound_dns

clickhouse
  └── base_docker
        └── base_packages

grafana, proxy
  └── (requires clickhouse + Docker already running)

base_pi, base_linux
  └── base_packages
  └── base_network

base_dev, base_samba, base_cups, pdf_server, telegraf, otel_collector
  └── base_packages

base_ssh, base_user, user_dotfiles   (no dependencies)
```

---

## Common operations

```bash
# Re-push SSH config everywhere
ansible-playbook playbooks/base-ssh.yml

# Update ClickHouse OTEL schema only (idempotent)
ansible-playbook playbooks/clickhouse-setup.yml --tags schema

# Update OTEL collector config only (no reinstall)
ansible-playbook playbooks/otel-setup.yml --tags configure

# Update Grafana dashboard only (no container restart unless changed)
ansible-playbook playbooks/grafana-setup.yml --tags provision

# Update nginx proxy config only (hot reload, no container restart)
ansible-playbook playbooks/proxy-setup.yml --tags configure

# Refresh internal DNS zone
ansible-playbook playbooks/unbound-setup.yml --tags internal-dns

# Check OS/arch/RAM on every host
ansible-playbook playbooks/system-info.yml
```

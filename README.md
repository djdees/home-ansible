# home-ansible

Single-repository Ansible configuration for the home network.
Consolidates the former `home_Ansible_roles`, `home_Ansible_playbooks`,
and `home_Ansible_inventories` repositories.

---

## Repository layout

```
home-ansible/
├── ansible.cfg                          ← roles_path, inventory, vault config
├── .yamllint                            ← YAML lint rules
│
├── inventories/
│   ├── home/
│   │   ├── hosts.yml                    ← all hosts (one source of truth)
│   │   └── group_vars/
│   │       ├── all/vars.yml             ← shared: timezone, gateway, domain, NTP
│   │       ├── all/vault.yml            ← symlink → ~/ansible/vault/all.yml
│   │       ├── pi_nodes/vars.yml        ← ansible_user/group for all Pis
│   │       ├── piholes/vars.yml         ← pihole vars
│   │       └── piholes/vault.yml        ← symlink → ~/ansible/vault/piholes.yml
│   └── test/
│       └── hosts.yml                    ← staging host(s)
│
├── roles/
│   ├── base_packages/                   ← deduplicates apt/git/python3
│   ├── base_linux/                      ← dist-upgrade for general Linux hosts
│   ├── base_pi/                         ← Pi locale, timezone, network (NM/dhcpcd/networkd), NTP
│   ├── user_dotfiles/                   ← bashrc, vimrc, shell selection
│   ├── base_user/                       ← creates djdees, passwordless sudo, SSH key
│   ├── base_ssh/                        ← templated sshd_config
│   ├── base_dev/                        ← Go, tmux, jq, yamllint, shellcheck
│   ├── base_docker/                     ← Docker CE (Debian repo URLs)
│   ├── base_samba/                      ← Samba file/print server
│   ├── base_cups/                       ← CUPS + HP drivers + Avahi AirPrint
│   ├── pdf_server/                      ← Apache2 PDF/media server
│   ├── telegraf/                        ← Telegraf metrics agent
│   ├── unbound_dns/                     ← Unbound recursive resolver
│   └── pihole_dns/                      ← Pi-hole v6 (depends on unbound_dns)
│
└── playbooks/
    ├── site.yml                         ← full bootstrap for all hosts
    ├── base-pi-setup.yml
    ├── base-linux-setup.yml
    ├── base-ssh.yml
    ├── base-dev.yml
    ├── base-docker.yml
    ├── unbound-setup.yml
    ├── pihole-setup.yml
    ├── dns-stack-setup.yml
    ├── samba-setup.yml
    ├── cups-setup.yml
    ├── pdf-server-setup.yml
    ├── telegraf-setup.yml
    ├── system-info.yml
    └── localhost-info.yml
```

---

## First-time setup

### 1. Clone and set up vault directory

```bash
git clone <your-repo-url> ~/ansible/home-ansible
cd ~/ansible/home-ansible

# Create vault directory outside the repo — never committed, never in zip
mkdir -p ~/ansible/vault
chmod 700 ~/ansible/vault

# Create encrypted vault files
ansible-vault create ~/ansible/vault/all.yml
# Add:
# vault_wifi_ssid: "YourSSID"
# vault_wifi_psk:  "YourPassword"

ansible-vault create ~/ansible/vault/piholes.yml
# Add:
# vault_pihole_webpassword: "YourPiholeAdminPassword"

# Create symlinks in group_vars pointing to the vault directory
ln -s ~/ansible/vault/all.yml \
  inventories/home/group_vars/all/vault.yml

ln -s ~/ansible/vault/piholes.yml \
  inventories/home/group_vars/piholes/vault.yml

# Optional: store vault password so you don't type it every run
echo 'your-vault-password' > ~/.ansible_vault_pass
chmod 600 ~/.ansible_vault_pass
# Then uncomment vault_password_file in ansible.cfg
```

### 2. Bootstrap a fresh Pi

Flash Raspberry Pi OS, run through initial setup, then:

```bash
# Enable SSH on the Pi
sudo systemctl enable ssh && sudo systemctl start ssh

# Add fixer sudoers entry on the Pi
sudo visudo /etc/sudoers.d/010_fixer-nopasswd
# Add: fixer ALL=(ALL) NOPASSWD: ALL

# Copy your SSH key to the Pi (replace with DHCP address)
ssh-copy-id -i ~/.ssh/id_ed25519.pub fixer@192.168.1.XXX

# First run — Pi is on DHCP, inventory has static IP
# -e ansible_host overrides connection address without affecting dhcpcd template
ansible-playbook playbooks/base-pi-setup.yml --limit piholes \
  -e ansible_host=192.168.1.XXX --ask-vault-pass

# After reboot at static IP, run DNS stack
ansible-playbook playbooks/dns-stack-setup.yml --ask-vault-pass
```

---

## Common operations

```bash
# Refresh internal DNS zone only
ansible-playbook playbooks/unbound-setup.yml --tags internal-dns --ask-vault-pass

# Re-push sshd_config across all hosts
ansible-playbook playbooks/base-ssh.yml --ask-vault-pass

# Check OS/arch/RAM on every host
ansible-playbook playbooks/system-info.yml --ask-vault-pass

# Run against test inventory
ansible-playbook playbooks/base-pi-setup.yml -i inventories/test/hosts.yml --ask-vault-pass
```

---

## User model

| Account | Purpose | Managed by |
|---|---|---|
| `fixer` | Ansible/admin — all playbook runs | Manual at imaging time |
| `djdees` | Personal interactive sessions | `base_user` role |

Both accounts are kept permanently. `fixer` is never removed.

---

## Network stack

```
LAN clients → Pi-hole (192.168.1.8:53) → Unbound (127.0.0.1:5335) → Root servers
                                        ↘ home.arpa zone (local)
```

- **Pi-hole v6** — ad blocking, query logging, upstream set to unbound
- **Unbound** — recursive resolution, DNSSEC validation, internal `.home.arpa` zone

### Verifying the stack

```bash
ssh fixer@192.168.1.8

# Unbound direct
dig @127.0.0.1 -p 5335 google.com
dig @127.0.0.1 -p 5335 pihole.home.arpa

# Through Pi-hole
dig @127.0.0.1 google.com
dig @127.0.0.1 pihole.home.arpa
dig @127.0.0.1 -x 192.168.1.8

# From another machine
dig @192.168.1.8 google.com
dig @192.168.1.8 pihole.home.arpa
```

---

## Role dependency graph

```
pihole_dns
  └── unbound_dns

base_pi, base_linux, base_dev, base_docker, base_samba, base_cups, pdf_server, telegraf
  └── base_packages

base_ssh      (no dependencies)
user_dotfiles (no dependencies)
base_user     (no dependencies)
```

---

## Network manager detection

`base_pi` auto-detects which network manager is active and uses the correct approach:

| Service | Config method | Used on |
|---|---|---|
| NetworkManager | `nmcli connection modify` | Bookworm, Trixie, Ubuntu |
| dhcpcd | `dhcpcd.conf` template | Bullseye and older |
| systemd-networkd | `.network` file template | Minimal server installs |

---

## Shell migration (bash → zsh)

Set `user_shell: zsh` in `group_vars/all/vars.yml` when ready.
The `user_dotfiles` role handles installation and `chsh` automatically.

---

## Secrets management

```bash
# View/edit vault files
ansible-vault view  ~/ansible/vault/all.yml
ansible-vault edit  ~/ansible/vault/piholes.yml

# Re-key to a new password
ansible-vault rekey ~/ansible/vault/all.yml
ansible-vault rekey ~/ansible/vault/piholes.yml
```

Vault files live in `~/ansible/vault/` — outside the repo, never committed.
Symlinks in `group_vars/` point to them so Ansible finds them normally.

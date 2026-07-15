# 3x-ui Multi-Server Ansible Setup

Automates deploying and configuring [3x-ui](https://github.com/MHSanaei/3x-ui)
(the Xray-core web panel) across multiple VPN servers, running each panel in
Docker with a unique admin port/path/password per host, firewall hardening,
and optional auto-provisioning of a default VLESS+Reality inbound.

## Project layout

```
3xui-ansible/
├── ansible.cfg
├── inventory/hosts.yml      # your servers go here
├── group_vars/all.yml       # shared settings
├── group_vars/vault.yml     # secrets (encrypt this!)
├── requirements.yml         # Galaxy collections needed
├── playbook.yml             # entrypoint
└── roles/
    ├── common/   # OS updates, swap, fail2ban, SSH hardening
    ├── docker/   # installs Docker Engine + Compose plugin
    └── xui/      # deploys 3x-ui container + configures panel + API provisioning
```

## 1. Install dependencies

```bash
pip install ansible
ansible-galaxy collection install -r requirements.yml
```

## 2. Edit your inventory

Update `inventory/hosts.yml` with your real server IPs, domains, and a
**unique `panel_webpath` per host** — don't reuse the same admin path across
servers.

## 3. Set your admin password

```bash
ansible-vault encrypt group_vars/vault.yml
ansible-vault edit group_vars/vault.yml   # to change the password later
```

## 4. Run it

```bash
ansible-playbook playbook.yml --ask-vault-pass
```

Target a single server:

```bash
ansible-playbook playbook.yml --ask-vault-pass --limit vpn-sg1
```

Skip the automatic inbound provisioning (just deploy the panel, configure
inbounds manually via the web UI):

```bash
ansible-playbook playbook.yml --ask-vault-pass -e provision_default_inbound=false
```

## What you get per server

- Docker + 3x-ui panel running with `network_mode: host`
- Admin panel reachable at `https://<domain>:<panel_port><panel_webpath>`
- (optional) a default VLESS + TCP + Reality inbound, with the Reality
  public key printed at the end of the run — you'll need it for client
  configs

## Notes / things to adjust for your environment

- **TLS for the panel itself**: this setup doesn't put a reverse proxy in
  front of the panel. For production, put Nginx/Caddy with a real
  Let's Encrypt cert in front of the panel port, or access it over the
  Reality-protected inbound only. Reality doesn't need a panel-facing cert
  since it disguises TLS handshakes to the `reality_dest` you configured.
- **Client configuration**: after a run, grab each server's inbound UUID and
  the printed Reality public key, then generate `vless://` links or QR codes
  with your client of choice (v2rayN, NekoBox, etc.). This can also be
  automated with another small role hitting `/panel/api/inbounds/list`.
- **Scaling to more servers**: just add more entries under
  `xui_servers.hosts` in the inventory — nothing else needs to change.
- **`serial: 1`** in the playbook rolls servers out one at a time so a bad
  config doesn't hit all of them simultaneously; remove it for faster
  parallel runs once you trust the playbook.

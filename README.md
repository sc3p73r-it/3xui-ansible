# 3x-ui Multi-Server Ansible Setup

Automates deploying and configuring [3x-ui](https://github.com/MHSanaei/3x-ui)
(the Xray-core web panel) across multiple VPN servers: each panel runs in
Docker with a unique admin port/path/password per host, base OS hardening
packages are installed, and a default VLESS+Reality inbound can be
auto-provisioned through the panel's REST API.

Target OS: **Debian/Ubuntu** (the `common` and `docker` roles use `apt`
directly, not a package-manager-agnostic module — RHEL/Alma/Rocky hosts are
not supported without changes).

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
    ├── common/   # apt update/upgrade, base packages, timezone, swapfile
    ├── docker/   # installs Docker Engine + Compose plugin
    └── xui/      # deploys 3x-ui container + configures panel + API provisioning
```

## How the run flows

`playbook.yml` targets the `xui_servers` group and applies three roles in
order, one host at a time (`serial: 1`):

1. **`common`** — `apt update && apt dist-upgrade`, installs base tooling
   (`curl`, `wget`, `unzip`, `socat`, `cron`, `ufw`, `fail2ban`,
   `ca-certificates`, `gnupg`, `lsb-release`, `python3-pip`), sets the server
   timezone, and creates + persists a 1G swapfile if one doesn't already
   exist (useful on small VPS instances).
2. **`docker`** — adds Docker's apt GPG key and repo, installs
   `docker-ce`, `docker-ce-cli`, `containerd.io`, the Buildx and Compose
   plugins, enables the `docker` service, and installs the Python `docker`
   and `requests` libraries via pip (required by the Ansible Docker
   modules used in the next role).
3. **`xui`** — the core of the project:
   - Creates `{{ xui_data_dir }}/db` and `/cert` on the host.
   - Templates a `docker-compose.yml` (host networking, bind-mounted
     `db/` and `cert/` volumes) and brings the container up.
   - Waits for the panel port, then uses the `x-ui` CLI **inside the
     container** to set the admin username, password, panel port, and
     web base path (`docker exec ... x-ui setting -username ... -password
     ... -port ... -webBasePath ...`), then restarts the container so the
     new settings take effect.
   - Optionally (`provision_default_inbound`, default `true`):
     generates a per-server x25519 keypair via `xray x25519`, generates a
     client UUID, logs into the panel's REST API to grab a session
     cookie, and POSTs to `/panel/api/inbounds/add` to create a default
     VLESS + TCP + Reality inbound. The Reality public key is printed at
     the end of the run — you'll need it to build client configs.

## Variables reference

| Variable | Where it's set | Purpose |
|---|---|---|
| `ansible_host`, `domain`, `server_location`, `panel_port`, `panel_webpath`, `xray_inbound_port` | `inventory/hosts.yml` (per host) | Per-server identity; **`panel_webpath` must be unique per host** |
| `xui_image`, `xui_container_name`, `xui_data_dir` | `group_vars/all.yml` | Container image/name and host data directory |
| `panel_username`, `panel_password` (→ `vault_panel_password`) | `group_vars/all.yml` / `group_vars/vault.yml` | Panel admin credentials |
| `default_protocol`, `default_flow`, `default_network`, `default_security`, `reality_dest`, `reality_server_names` | `group_vars/all.yml` | Default inbound shape when auto-provisioning |
| `server_timezone` | **not set anywhere in this repo** | Passed to `community.general.timezone` — see Known issues below |
| `provision_default_inbound` | extra-var only (defaults to `true` in the role) | Set to `false` to skip API provisioning |

## 1. Install dependencies

```bash
pip install ansible
ansible-galaxy collection install -r requirements.yml
```

Requires the `community.general`, `community.docker`, and `ansible.posix`
collections (pinned minimum versions in `requirements.yml`).

## 2. Edit your inventory

Update `inventory/hosts.yml` with your real server IPs, domains, and a
**unique `panel_webpath` per host** — don't reuse the same admin path across
servers.

## 3. Set your admin password and timezone

```bash
ansible-vault encrypt group_vars/vault.yml
ansible-vault edit group_vars/vault.yml   # to change the password later
```

`group_vars/vault.yml` ships with a placeholder password
(`vault_panel_password: "Password123!"`) — change it before encrypting, and
never commit the unencrypted file.

Also add a `server_timezone` value (e.g. `Etc/UTC`) somewhere in
`group_vars/all.yml` or per-host — the `common` role references it but no
default currently exists in this repo, so a run will fail on the timezone
task until it's defined.

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

## Known issues / gaps to be aware of

- **`server_timezone` is undefined.** `roles/common/tasks/main.yml` sets
  the timezone from `{{ server_timezone }}`, but no default is defined in
  `group_vars/all.yml` or the inventory. Add one before running, or the
  play will fail on that task.
- **`ufw` and `fail2ban` are installed but not configured.** The `common`
  role only installs these packages — it doesn't enable `ufw`, open/deny
  any ports, or write a `fail2ban` jail. "Firewall hardening" here means
  the tooling is present, not that a firewall policy is enforced. Add
  explicit `ufw` allow/enable tasks (SSH, `panel_port`,
  `xray_inbound_port`) if you need this enforced.
- **No SSH hardening tasks exist** despite what earlier docs implied —
  there's nothing in `common` that touches `sshd_config`. Handle key-only
  auth / root login policy separately if required.
- **No reverse proxy / TLS in front of the panel.** See the note below.

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

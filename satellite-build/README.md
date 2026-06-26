# satellite-build

Ansible role that installs **Red Hat Satellite 6.19** on RHEL 9 and configures it end-to-end: storage, firewall, subscription, installer run, manifest import, repositories, sync plans, content views, lifecycle environments, promotion, and activation keys.

The goal is a single playbook run that takes a freshly provisioned RHEL 9 VM and leaves you with a Satellite server that's ready for hosts to register against.

## What this role does (task by task)

| File | Purpose |
|---|---|
| `01-prechecks.yml` | Verifies CPU, RAM, swap, and an unused disk meet minimums. Skips the host (not fails — `meta: end_host`) if any check fails. |
| `02-prechecks.yml` | Creates mount points, installs blivet/lvm tooling, sets the hostname, ensures `/etc/hosts` has forward + reverse DNS entries on both the Satellite host and the controller. |
| `03-create_vg.yml` | Builds VG `vgsat` on the selected disk via `redhat.rhel_system_roles.storage` and carves out LVs for `/var/log`, `/var/lib/pgsql`, `/opt/puppetlabs`, `/var/lib/pulp`. |
| `04-firewall_hosts_resolve.yml` | Opens required ports + services on firewalld, enforces SELinux, sets locale to `en_US.UTF-8`. |
| `05-redhat_subscription.yml` | Registers the host with RHSM via `redhat.rhel_system_roles.rhc`, disables all repos, enables only the four required for Satellite 6.19 on RHEL 9, runs a full `dnf upgrade`. |
| `06-satellite-install.yml` | Installs the `satellite` package and runs `satellite-installer` via `redhat.satellite_operations.installer`. Includes a log-based verification phase that distinguishes "actually failed" from "Ansible timed out but install finished". |
| `07-configure-satellite.yml` | Imports the manifest, enables + syncs RHEL 8/9/10 BaseOS/AppStream and Satellite Client 6 repos, creates a daily sync plan, lifecycle environments (Dev/QA/Prod), content views and composite content views, promotes them through the lifecycle, and creates activation keys. |

## Requirements

### Target host (Satellite server)
- RHEL 9
- Minimum 4 CPU cores
- Minimum 20 GB RAM
- Minimum 4 GB swap
- Minimum 400 GB of unused, unpartitioned disk space (the role picks the first disk that meets the size threshold and has no partitions)
- A unique hostname (the role will set it from `host_name` in `vars/main.yml` — adjust before running)
- A working dnf repo. Task 02 installs `python3-blivet`, `libblockdev-lvm`, and `lvm2`, which task 03 depends on for LVM operations. If those packages can't be fetched, task 03 will fail.

### Controller
- Ansible 2.15+ recommended
- Python 3.9+
- The collections listed in [Dependencies](#dependencies) installed
- HashiCorp Vault reachable from the controller (see [Vault setup](#vault-setup))
- The Red Hat subscription manifest downloaded from <https://console.redhat.com/> and placed in `files/` (see [Manifest](#manifest))

## Vault setup

This role pulls all credentials from HashiCorp Vault via the `community.hashi_vault` lookup plugin. The role does not bundle a Vault setup — you need a running Vault that the controller can reach, with the right paths and policy in place.

### One-time install + persistent Vault on the controller

If you don't already have a Vault instance and want one running locally on the Ansible controller, the persistent (file-backed) setup is:

```bash
# 1. Install (dnf5 / Fedora syntax — adjust if on RHEL/dnf4 with --add-repo)
sudo dnf install -y dnf-plugins-core
sudo dnf config-manager addrepo --from-repofile=https://rpm.releases.hashicorp.com/fedora/hashicorp.repo
sudo dnf install -y vault

# 2. Config + storage dir
mkdir -p ~/.vault/data
cat > ~/.vault/vault.hcl <<EOF
storage "file" {
  path = "/home/${USER}/.vault/data"
}
listener "tcp" {
  address     = "0.0.0.0:8200"
  tls_disable = 1
}
api_addr = "http://127.0.0.1:8200"
ui       = true
EOF

# 3. First-time init (in a second terminal once vault is running)
export VAULT_ADDR='http://127.0.0.1:8200'
vault server -config=$HOME/.vault/vault.hcl &
vault operator init -key-shares=1 -key-threshold=1
# Save the printed Unseal Key and Initial Root Token — they cannot be recovered.

# 4. Unseal (required after every restart in server mode)
vault operator unseal <unseal-key>
export VAULT_TOKEN='<root-token>'

# 5. Convenience start+unseal script
cat > ~/.vault/start-vault.sh <<'EOF'
#!/bin/bash
export VAULT_ADDR='http://127.0.0.1:8200'
vault server -config=$HOME/.vault/vault.hcl &
sleep 2
vault operator unseal <your-unseal-key>
echo "Vault is running and unsealed at http://localhost:8200"
EOF
chmod +x ~/.vault/start-vault.sh
```

> WSL note: on WSL, `~/.vault` being a directory will make `vault login` fail with `is a directory`. Skip `vault login` and just `export VAULT_TOKEN` instead — Vault CLI reads the token from the env var.

### AppRole + policy + KV engine

```bash
# Enable AppRole and the KV v2 engine at the path this role expects
vault auth enable approle
vault secrets enable -path=secret -version=2 kv

# Policy: read-only access under secret/larry_inc/*
cat > /tmp/larry-read-secrets.hcl <<EOF
path "secret/data/larry_inc/*"      { capabilities = ["read"] }
path "secret/metadata/larry_inc/*"  { capabilities = ["read", "list"] }
EOF
vault policy write larry-read-secrets /tmp/larry-read-secrets.hcl

# Role bound to that policy
vault write auth/approle/role/larry-role secret_id_ttl=0 secret_id_num_uses=0
vault write auth/approle/role/larry-role token_policies="larry-read-secrets"

# Capture credentials — these go into the controller's env
vault read  auth/approle/role/larry-role/role-id
vault write -f auth/approle/role/larry-role/secret-id
```

### Secrets the role expects to exist

Create these under `secret/larry_inc/` (UI or CLI):

| Path | Keys | Used in |
|---|---|---|
| `secret/larry_inc/redhat_cred` | `rhel_user`, `rhel_pass` | Task 05 (RHSM registration) |
| `secret/larry_inc/satellite`   | `satellite_username`, `satellite_password` | Task 06 (installer) + all of task 07 |

CLI example:
```bash
vault kv put secret/larry_inc/redhat_cred rhel_user='<rh-login>' rhel_pass='<rh-pass>'
vault kv put secret/larry_inc/satellite  satellite_username='admin' satellite_password='<choose-one>'
```

### Environment variables on the controller

The role reads these via `lookup('env', ...)`:

```bash
export VAULT_ADDR='http://127.0.0.1:8200'   # client URL — NOT 0.0.0.0
export VAULT_ROLE_ID='<role-id from above>'
export VAULT_SECRET_ID='<secret-id from above>'
```

Persist them in `~/.zshrc` (or `~/.bashrc`) so every new shell has them. `VAULT_TOKEN` is not required by the role — AppRole handles auth.

## Manifest

Download your subscription manifest from <https://console.redhat.com/> → Subscriptions → Manifests, and drop it into the role's `files/` directory:

```
files/larry-sat-manifest.zip
```

The path is referenced by `satellite_manifest_path` in `vars/main.yml` using `{{ role_path }}/files/...`, so the role works regardless of where it's vendored. If you rename the manifest file, update that variable to match.

## Role Variables

All variables live in `vars/main.yml`. Key ones to review before running:

| Variable | Default | What it controls |
|---|---|---|
| `host_name` | `larrysat.lab.example.com` | FQDN to set on the Satellite host |
| `min_cpu_cores` / `min_ram_gb` / `min_disk_gb` / `min_swap_gb` | 4 / 20 / 400 / 4 | Precheck thresholds |
| `mount_points` | `/var/lib/pulp`, `/var/lib/pgsql`, `/opt/puppetlabs` | Directories created for LV mounts |
| `ports`, `services` | see vars | firewalld ports/services |
| `satellite_initial_location` | `Melton` | Initial Location passed to `satellite-installer` |
| `satellite_organization` | `Larry_inc` | Target organization name in Satellite |
| `satellite_server_url` | `https://{{ ansible_fqdn }}` | Used by all `redhat.satellite.*` modules |
| `satellite_manifest_path` | `{{ role_path }}/files/larry-sat-manifest.zip` | Path to the downloaded manifest |
| `satellite_products` | see vars | Repository sets to enable via `redhat.satellite.repositories` |
| `satellite_sync_list` | see vars | Repos to sync (must match post-enable names exactly) |
| `satellite_lifecycle_environments` | `Dev → QA → Prod` | Lifecycle path after Library |
| `satellite_username` / `satellite_password` | vault lookup | Pulled from `secret/larry_inc/satellite` |
| `rhc_auth.login.username` / `.password` | vault lookup | Pulled from `secret/larry_inc/redhat_cred` |

## Dependencies

Install the following collections on the controller before running:

```bash
ansible-galaxy collection install \
  redhat.satellite \
  redhat.satellite_operations \
  redhat.rhel_system_roles \
  ansible.posix \
  community.general \
  community.hashi_vault
```

## Example Playbook

```yaml
---
- name: Install and configure Satellite
  hosts: larry-sat
  vars:
    ansible_ssh_pipelining: true
  roles:
    - satellite-build
```

## Useful tags

Run a subset of the role with `--tags`:

| Tag | Scope |
|---|---|
| `cpu`, `ram`, `swap`, `disk` | Individual precheck families in 01 |
| `checks` | All of 02 (mount points, blivet, hostname, DNS) |
| `lvm` | 03 (storage role) |
| `firewalld` | 04 (firewalld, ping, locale) |
| `subscription` | 05 (RHSM register + repos + upgrade) |
| `manifest` | Manifest import step in 07 |
| `repos` | Repositories + sync + sync plan in 07 |
| `content_views` | Content view creation in 07 |
| `promote` | Composite CV promotion in 07 |
| `ak-keys` | Activation key creation in 07 |

## Recommended workflow for first run

1. Snapshot the Satellite VM.
2. Run the playbook end-to-end.
3. If it fails partway, fix the underlying issue, revert the snapshot, re-run.
4. Repeat until a clean start-to-finish run.

Satellite installs are not gracefully restartable from arbitrary failure points — partial state (storage already carved, partial sync, half-promoted CVs) is harder to clean up than reverting and re-running.

## Author

Larry Akinnawonu

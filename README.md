# Ansible Server Automation Project

A self-directed hands-on project: configuring a small fleet of Linux servers with Ansible, from first password login to a monitored, rolling-updated service.

The three "servers" are Docker containers running real systemd and SSH, so Ansible treats them like ordinary VMs. Everything runs on one machine.

## What it does

| Playbook | What it does |
|---|---|
| `bootstrap.yml` | First contact: logs in with a password and installs an SSH key, so every later run is key-based. |
| `site.yml` | Base setup for all servers: packages, admin and deploy users, SSH hardening (validated with `sshd -t` before it is applied). Web servers get an app config rendered from a Jinja2 template; DB servers get their own packages. |
| `monitoring.yml` | Applies the `node_exporter` role one server at a time (`serial: 1`), stops on the first failure (`max_fail_percentage: 0`) and health-checks `/metrics` after each server. |
| `prometheus-targets.yml` | Generates the Prometheus scrape config from the inventory, so the inventory is the single source of truth for what gets monitored. Restarts Prometheus only when the config changes. |

### The `node_exporter` role

- Dedicated system user with no login shell; the service never runs as root.
- Release downloaded once on the control node and verified against the published `sha256sums.txt`, then pushed to the servers (works for servers without internet access).
- Idempotent unpack (`creates:`), binary installed to `/usr/local/bin`.
- systemd unit from a template; a handler restarts the service only when the binary or the unit changed.
- Version, architecture, port and download URL are role defaults, so an upgrade is one variable:
  `ansible-playbook -i inventory.ini monitoring.yml -e node_exporter_version=1.12.1`
- Tags `user`, `install`, `service` for partial runs (`--tags service`).

### Also covered

- Inventory groups (`web`, `db`, parent group `lab`), `group_vars` / `host_vars` with per-host overrides.
- Loops, conditionals on facts, registered variables, magic variables (`groups`, `hostvars`) in templates.
- Secrets in `ansible-vault`, referenced from plain variables (`grafana_admin_password` → `vault_grafana_admin_password`).
- Drift detection with `--check --diff`.
- Passes `ansible-lint --profile production`.

## Layout

```
inventory.ini              hosts and groups
bootstrap.yml              password -> SSH key
site.yml                   base setup, web and db plays
monitoring.yml             rolling node_exporter rollout + health check
prometheus-targets.yml     Prometheus targets from the inventory
group_vars/, host_vars/    variables by group and host (vault.yml is encrypted)
templates/                 app.conf.j2, prometheus.yml.j2
roles/node_exporter/       the role
setup/                     Dockerfile + compose for the three test servers
```

## Run it

Requirements: Docker with Compose, Ansible (tested with ansible-core 2.21), `sshpass` for the first password login, and the `ansible.posix` collection.

```bash
ansible-galaxy collection install ansible.posix
ssh-keygen -t ed25519 -f ~/.ssh/ansible_lab -N ''

cd setup && docker compose up -d --build && cd ..      # web1, web2, db1

# group_vars/all/vault.yml is encrypted with my password: replace it with your own
echo 'vault_grafana_admin_password: change-me' > group_vars/all/vault.yml
ansible-vault encrypt group_vars/all/vault.yml

ansible-playbook -i inventory.ini bootstrap.yml -k --ask-vault-pass   # SSH password: ansible
ansible-playbook -i inventory.ini site.yml --ask-vault-pass
ansible-playbook -i inventory.ini monitoring.yml --ask-vault-pass
curl -s 127.0.0.1:9101/metrics | grep node_exporter_build_info
```

Every command needs the vault password, because `group_vars/all` is loaded for every host. To skip the prompt, put the password in a file outside the repo (`chmod 600`) and set `ANSIBLE_VAULT_PASSWORD_FILE` to its path.

### Connecting to Prometheus

This project plugs into my [Prometheus monitoring project](https://github.com/LIEFAKE/prometheus-monitoring-project). With that project cloned locally:

1. In its `docker-compose.yml`, attach the `prometheus` service to a second network, `ansible-lab`, declared at the top level with `external: true` (it is created by `setup/` here).
2. Generate its scrape config from this inventory and restart Prometheus:

```bash
ansible-playbook -i inventory.ini prometheus-targets.yml -e prometheus_project_dir=/path/to/prometheus-monitoring-project --ask-vault-pass
```

Prometheus then scrapes `web1:9100`, `web2:9100`, `db1:9100` by container name, and the Grafana "Node Exporter Full" dashboard shows all three.

### Notes

- The test servers are privileged containers with systemd. On WSL2 the image masks `systemd-binfmt` so the containers don't break Windows interop on the host.
- SSH host key checking is turned off in the inventory because the containers are recreated often. Don't do that with real servers.

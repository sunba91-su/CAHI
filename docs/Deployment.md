## Deploying CAPE using CAHI

### Prerequisites

- Ubuntu 22.04/24.04 LTS target host(s)
- Python 3.10+ on the control node
- Ansible 2.12+ installed
- SSH access to target host(s) as root
- SSH key-based authentication configured

### Directory Structure

```
CAHI/
├── ansible.cfg                    # Ansible configuration
├── deploy.sh                      # Deployment script
├── inventory/
│   ├── hosts.ini                  # Inventory file
│   └── group_vars/
│       ├── all/
│       │   ├── vars.yml           # Global variables
│       │   └── vault.yml          # Encrypted secrets
│       ├── cape_host.yml          # Host-specific vars
│       ├── cape_web.yml           # Web node vars
│       ├── cape_worker.yml        # Worker node vars
│       ├── cape_guests.yml        # Guest VM vars
│       └── cape_guests_winrm.yml  # WinRM connection vars
├── playbooks/                     # Main playbooks
└── roles/                         # Ansible roles
    ├── dependencies/              # System packages
    ├── kvm/                       # KVM/QEMU/libvirt
    ├── nginx/                     # Reverse proxy
    ├── postgres/                  # PostgreSQL database
    ├── mongodb/                   # MongoDB (optional)
    ├── suricata/                  # IDS/IPS
    ├── yara/                      # YARA rules
    ├── tor/                       # Tor proxy
    ├── guest_provisioning/        # Windows VM provisioning
    ├── cape/                      # CAPE application
    ├── fail2ban/                  # Intrusion prevention
    ├── logrotate/                 # Log rotation
    ├── clamav/                    # Antivirus
    ├── capa/                      # CAPA analysis
    ├── volatility/                # Memory forensics
    ├── gperftools/                # Performance tools
    ├── mfactor/                   # Memory factor
    ├── time/                      # Time synchronization
    ├── maxmind/                   # GeoIP database
    ├── cis/                       # CIS benchmarks
    ├── nginx_hardening/           # nginx security
    ├── suricata_update/           # Suricata rules
    ├── yara_update/               # YARA rules update
    ├── postgres_backup/           # Database backups
    └── log_analytics/             # Log aggregation
```

### Single-Node Deployment

A single-node deployment runs all CAPE components on one host.

```bash
# 1. Configure inventory
cp inventory/group_vars/all/vault.yml inventory/group_vars/all/vault.yml.bak
vim inventory/hosts.ini

# 2. Set secrets
vim inventory/group_vars/all/vault.yml

# 3. Deploy
./deploy.sh
```

The `deploy.sh` script will:
1. Validate the inventory
2. Run the playbook with vault password
3. Display deployment summary

### Multi-Node Deployment

For production environments, distribute components across multiple hosts:

| Component | Host Group | Description |
|-----------|------------|-------------|
| Control | `[cape_control]` | PostgreSQL, Redis, nginx, API |
| Web | `[cape_web]` | Web interface, gunicorn |
| Worker | `[cape_worker]` | KVM, CAPE analysis, suricata |
| Guests | `[cape_guests]` | Windows VMs (optional) |

#### Step 1: Configure Inventory

```ini
# inventory/hosts.ini
[cape_control]
control-node ansible_host=192.168.1.10

[cape_web]
web-node ansible_host=192.168.1.11

[cape_worker]
worker-node ansible_host=192.168.1.12

[cape_guests]
guest-1 ansible_host=192.168.1.20 libvirt_name=guest-1 guest_ip=192.168.1.20
```

#### Step 2: Deploy Control Node

```bash
ansible-playbook playbooks/multi-node-control.yml -i inventory/hosts.ini --ask-vault-pass
```

#### Step 3: Deploy Web Node

```bash
ansible-playbook playbooks/multi-node-web.yml -i inventory/hosts.ini --limit web -K
```

#### Step 4: Deploy Worker Node(s)

```bash
ansible-playbook playbooks/multi-node-worker.yml -i inventory/hosts.ini --limit worker -K
```

#### Step 5: Register Worker

```bash
ansible-playbook playbooks/register-worker.yml -i inventory/hosts.ini --ask-vault-pass
```

### Configuration Variables

Key variables in `inventory/group_vars/all/vars.yml`:

| Variable | Default | Description |
|----------|---------|-------------|
| `deployment_mode` | `single` | Deployment topology: `single` or `multi` |
| `cape_user` | `cape` | System user for CAPE services |
| `cape_main_dir` | `/opt` | CAPE installation directory |
| `network_iface` | `virbr1` | Internal bridge interface |
| `iface_ip` | `192.168.1.1` | Host IP on internal bridge |
| `db_host` | `localhost` | PostgreSQL host |
| `mongo_enable` | `false` | Enable MongoDB (optional) |
| `use_uv` | `false` | Use uv instead of poetry |

### Secrets

Encrypt `inventory/group_vars/all/vault.yml` with:

```bash
ansible-vault encrypt inventory/group_vars/all/vault.yml
```

Required secrets:
- `vault_db_password`: PostgreSQL password
- `vault_mongo_pass`: MongoDB password
- `vault_tor_password`: Tor control password

### Verification

After deployment, verify the installation:

```bash
ansible-playbook playbooks/verify.yml -i inventory/hosts.ini
```

### Smoke Tests

Run automated smoke tests:

```bash
ansible-playbook playbooks/smoke-test.yml -i inventory/hosts.ini
```

### Troubleshooting

#### Check Service Status

```bash
systemctl status cape cape-web cape-processor cape-rooter
```

#### View Logs

```bash
journalctl -u cape -f
journalctl -u cape-web -f
```

#### Common Issues

1. **Vault password error**: Ensure `.vault_pass` exists or use `--ask-vault-pass`
2. **Permission denied**: Run with `-K` for become password
3. **SSH connection failed**: Check `inventory/hosts.ini` and SSH keys
4. **Service won't start**: Check `/var/log/cape/` for error logs

# HashiCorp Vault HA Cluster with Ansible

This repository contains Ansible code to deploy a 3-node HashiCorp Vault HA cluster using Podman containers managed by SystemD.

## Architecture

- 3 Vault nodes (vault1, vault2, vault3) running on Rocky Linux VMs
- Raft storage backend for HA clustering
- Podman containers managed by SystemD services
- Self-signed certificates for TLS
- Single unseal key stored on filesystem with automatic unsealing

## Files Structure

```
├── Vagrantfile               # Vagrant configuration for VMs
├── inventory/
│   ├── hosts.yml            # Ansible inventory (production)
│   └── vagrant-hosts.yml    # Ansible inventory (vagrant)
├── playbooks/
│   ├── deploy-vault.yml     # Main deployment playbook
│   └── vault-operations.yml # Service operations (start/stop/reset)
└── templates/
    ├── vault.hcl.j2        # Vault configuration
    ├── podman-compose.yml.j2 # Podman compose file
    ├── vault.service.j2     # SystemD service file
    └── vault-unseal.sh.j2   # Auto-unseal script
```

## Quick Start with Vagrant

### Step 1: Spin up the VMs

```bash
# Start all 3 VMs
vagrant up

# Check VM status
vagrant status

# SSH into individual VMs (if needed)
vagrant ssh vault1
vagrant ssh vault2
vagrant ssh vault3
```

The VMs will be available at:
- vault1: 192.168.56.11
- vault2: 192.168.56.12
- vault3: 192.168.56.13

### Step 2: Deploy Vault with Ansible

```bash
# Deploy the Vault cluster
ansible-playbook -i inventory/vagrant-hosts.yml playbooks/deploy-vault.yml

# Optional: Enable unseal key logging during deployment
ansible-playbook -i inventory/vagrant-hosts.yml playbooks/deploy-vault.yml -e vault_log_unseal_key=true
```

### Step 3: Access Vault Web UI

After deployment, you can access the Vault Web UI at:
- **Primary UI**: http://localhost:8200 (forwarded from vault1)
- **Direct access**: https://192.168.56.11:8200 (vault1)

**Login credentials:**
- Root token is stored in `/opt/vault/config/root.token` on each VM
- Retrieve it with: `vagrant ssh vault1 -c "sudo cat /opt/vault/config/root.token"`

## SSH Access

You can SSH directly to the VMs using:
```bash
ssh vagrant@192.168.56.11  # vault1
ssh vagrant@192.168.56.12  # vault2
ssh vagrant@192.168.56.13  # vault3
```

## Production Usage

For production deployments, use the main inventory file:
```bash
ansible-playbook -i inventory/hosts.yml playbooks/deploy-vault.yml
```

## Lifecycle Management

### Starting and Stopping VMs

```bash
# Stop all VMs (preserves state)
vagrant halt

# Start all VMs
vagrant up

# Restart all VMs
vagrant reload

# Check VM status
vagrant status
```

### Vault Service Operations

```bash
# Check Vault service status
ansible-playbook -i inventory/vagrant-hosts.yml playbooks/vault-operations.yml -e vault_operation=status

# Start Vault services
ansible-playbook -i inventory/vagrant-hosts.yml playbooks/vault-operations.yml -e vault_operation=start

# Stop Vault services
ansible-playbook -i inventory/vagrant-hosts.yml playbooks/vault-operations.yml -e vault_operation=stop

# Restart Vault services
ansible-playbook -i inventory/vagrant-hosts.yml playbooks/vault-operations.yml -e vault_operation=restart
```

### Re-deploying After VM Restart

The setup is designed to be idempotent. After stopping and starting VMs, simply re-run the deployment:

```bash
# VMs stopped and started - re-deploy to ensure everything is running
ansible-playbook -i inventory/vagrant-hosts.yml playbooks/deploy-vault.yml
```

This will:
- Skip initialization if already done
- Reuse existing unseal keys and certificates  
- Start services if they're not running
- Unseal Vault automatically

## Complete Reset and Fresh Start

### Option 1: Reset Vault Cluster Data Only

To wipe all Vault data but keep VMs and certificates:

```bash
# Reset Vault cluster (removes all vault data, keeps VMs)
ansible-playbook -i inventory/vagrant-hosts.yml playbooks/vault-operations.yml -e vault_operation=reset

# Re-initialize with fresh cluster
ansible-playbook -i inventory/vagrant-hosts.yml playbooks/deploy-vault.yml
```

### Option 2: Complete VM Reset

To completely start over with fresh VMs:

```bash
# Destroy all VMs and data
vagrant destroy -f

# Start fresh VMs
vagrant up

# Deploy fresh Vault cluster
ansible-playbook -i inventory/vagrant-hosts.yml playbooks/deploy-vault.yml
```

### Option 3: Selective VM Reset

To reset individual VMs:

```bash
# Destroy and recreate specific VM
vagrant destroy vault1 -f
vagrant up vault1

# Re-run deployment to restore cluster
ansible-playbook -i inventory/vagrant-hosts.yml playbooks/deploy-vault.yml
```

## Key Features

- **Idempotent Operations**: Can safely re-run deployment multiple times
- **Persistent State**: Survives VM stop/start cycles
- **Easy Reset**: Multiple options for clean slate restart
- **Automatic Initialization**: First node initializes Vault with a single unseal key
- **Key Distribution**: Unseal key is distributed to all nodes
- **Auto-Unsealing**: Vault automatically unseals on boot via SystemD
- **TLS Security**: Self-signed certificates for secure communication
- **HA Clustering**: Raft consensus for high availability

## Important Files

- `/opt/vault/config/unseal.key` - Single unseal key (mode 600)
- `/opt/vault/config/root.token` - Root token (mode 600)
- `/opt/vault/config/vault.hcl` - Vault configuration
- `/etc/systemd/system/vault.service` - SystemD service

## Security Notes

- Unseal keys are stored in filesystem with restricted permissions
- TLS certificates are auto-generated for secure communication
- Services run under dedicated vault user account
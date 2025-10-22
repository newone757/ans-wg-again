# WireGuard Ansible Setup Guide

## Project Structure

```
wireguard-ansible/
├── playbook.yml
├── inventory.yml
├── group_vars/
│   └── wireguard_servers/
│       └── vault.yml
├── templates/
│   └── wg0.conf.j2
├── ansible.cfg
└── wireguard-keys/          # Created automatically - backed up keys
    ├── origin1/
    │   ├── privatekey
    │   └── publickey
    ├── frontend1/
    │   ├── privatekey
    │   └── publickey
    └── ...
```

## Network Topology

This playbook creates a **hub-and-spoke** WireGuard network:

- **Origin/Backend Servers**: Can communicate with ALL servers (frontend + other origin)
- **Frontend Servers**: Can ONLY communicate with origin servers
- Frontend servers cannot see or communicate with each other

### Example Network Layout:
```
Origin Servers: 10.0.0.1-10
Frontend Servers: 10.0.0.11-254

Frontend1 (10.0.0.11) <---> Origin1 (10.0.0.1)
Frontend2 (10.0.0.12) <---> Origin1 (10.0.0.1)
Frontend3 (10.0.0.13) <---> Origin2 (10.0.0.2)
                              ^
                              |
                            Origin2 (10.0.0.2)
```

## Setup Instructions

### 1. Create the Project Directory Structure

```bash
mkdir -p wireguard-ansible/{group_vars/wireguard_servers,templates}
cd wireguard-ansible
```

### 2. Create Ansible Configuration

Create `ansible.cfg`:

```ini
[defaults]
inventory = inventory.yml
host_key_checking = False
retry_files_enabled = False
```

### 3. Create the Vault File

Create a single vault file to store all your unique sudo passwords:

```bash
ansible-vault create group_vars/wireguard_servers/vault.yml
```

You'll be prompted to create a vault password. Then add all your host-specific become passwords:

```yaml
---
vault_become_passwords:
  origin1: "origin1_sudo_password_here"
  origin2: "origin2_sudo_password_here"
  frontend1: "frontend1_sudo_password_here"
  frontend2: "frontend2_sudo_password_here"
  frontend3: "frontend3_sudo_password_here"
```

This way, you only manage one vault file but each host gets its own unique password!

### 4. Update the Inventory File

Edit `inventory.yml` with your actual server details:
- Replace IP addresses with your VPS IPs
- Update SSH ports
- Update admin usernames
- Update SSH key file paths
- Assign WireGuard IPs (origin: 10.0.0.1-10, frontend: 10.0.0.11+)
- Assign WireGuard ports (must be unique per host)

### 5. Create the Template Directory

Place the `wg0.conf.j2` template in the `templates/` directory.

### 6. Run the Playbook

**Dry run (check mode):**
```bash
ansible-playbook playbook.yml --ask-vault-pass --check
```

**Execute the playbook:**
```bash
ansible-playbook playbook.yml --ask-vault-pass
```

**Or use a vault password file:**
```bash
echo "your_vault_password" > .vault_pass
chmod 600 .vault_pass
ansible-playbook playbook.yml --vault-password-file .vault_pass
```

### 7. Verify the Configuration

After running the playbook, verify WireGuard is working:

```bash
# Check WireGuard status on each server
ansible wireguard_servers -m shell -a "wg show" --ask-vault-pass

# Test connectivity from frontend to origin
ansible frontend_servers -m shell -a "ping -c 3 10.0.0.1" --ask-vault-pass

# Verify frontend servers cannot reach each other (should fail)
# SSH to frontend1 and try: ping 10.0.0.12
```

## Managing the Vault

**Edit the vault:**
```bash
ansible-vault edit group_vars/wireguard_servers/vault.yml
```

**Change vault password:**
```bash
ansible-vault rekey group_vars/wireguard_servers/vault.yml
```

**View vault contents:**
```bash
ansible-vault view group_vars/wireguard_servers/vault.yml
```

## Adding New Servers

1. Add the new server to `inventory.yml` under the appropriate group (origin_servers or frontend_servers)
2. Run the playbook again:
```bash
ansible-playbook playbook.yml --ask-vault-pass
```

The playbook is idempotent and will:
- Configure the new server
- Update all existing servers with the new peer information

## Troubleshooting

**Check WireGuard interface status:**
```bash
ansible wireguard_servers -m shell -a "ip addr show wg0" --ask-vault-pass
```

**Check WireGuard service status:**
```bash
ansible wireguard_servers -m shell -a "systemctl status wg-quick@wg0" --ask-vault-pass
```

**View WireGuard logs:**
```bash
ansible wireguard_servers -m shell -a "journalctl -u wg-quick@wg0 -n 50" --ask-vault-pass
```

**Test specific connectivity:**
```bash
# From ansible control machine
ansible frontend1 -m shell -a "ping -c 3 10.0.0.1" --ask-vault-pass
```

## Security Notes

- Each server generates its own private/public key pair
- Private keys never leave their respective servers during generation
- Keys are backed up to `./wireguard-keys/` on your control machine with restrictive permissions
- **IMPORTANT**: Add `wireguard-keys/` to your `.gitignore` if using version control
- SSH keys for each server should be unique and properly secured
- The vault file contains sudo passwords - keep it secure
- WireGuard ports are opened in UFW automatically
- IP forwarding is enabled for routing between peers

## Network Architecture Benefits

This hub-and-spoke topology provides:
- **Isolation**: Frontend servers cannot communicate with each other
- **Security**: Origin servers control all traffic flow
- **Scalability**: Easy to add new frontend servers
- **Simplicity**: Fewer peer connections to manage
- **Control**: All frontend traffic must go through origin servers
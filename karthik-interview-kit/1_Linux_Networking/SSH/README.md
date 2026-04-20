# SSH - Secure Shell

## What is it used for?
SSH (Secure Shell) is a protocol for secure remote access:
- **Remote login**: Connect securely to remote servers
- **Secure file transfer**: Transfer files safely (SCP, SFTP)
- **Command execution**: Run commands on remote machines
- **Tunneling**: Create secure tunnels for network traffic
- **Git operations**: Authenticate with git repositories
- **Agent forwarding**: Forward SSH credentials securely

## Installation

```bash
# Linux
apt-get install openssh-client openssh-server  # Debian/Ubuntu
yum install openssh-clients openssh-server     # RHEL/CentOS
systemctl start ssh                            # Start SSH service

# macOS
# Pre-installed, enable in System Preferences > Sharing

# Windows
# Built-in (Windows 10+), or use PuTTY/Git Bash
```

## Basic Commands & Troubleshooting

### Essential Commands
```bash
# Connect to remote server
ssh user@hostname
ssh user@192.168.1.1

# Connect on non-standard port
ssh -p 2222 user@hostname

# Generate SSH key pair
ssh-keygen -t rsa -b 4096 -f ~/.ssh/id_rsa
ssh-keygen -t ed25519 -f ~/.ssh/id_ed25519

# Copy public key to server (password-less login)
ssh-copy-id -i ~/.ssh/id_rsa.pub user@hostname

# Copy files from remote to local
scp user@hostname:/remote/path /local/path

# Copy files from local to remote
scp /local/path user@hostname:/remote/path

# Run command on remote server
ssh user@hostname "command"

# Port forwarding (local)
ssh -L local_port:localhost:remote_port user@hostname

# Port forwarding (remote)
ssh -R remote_port:localhost:local_port user@hostname

# Add key to SSH agent
ssh-add ~/.ssh/id_rsa

# List loaded keys
ssh-add -l

# Test SSH connection with verbose output
ssh -v user@hostname
```

### Common Issues & Resolution

**Issue: Permission Denied (publickey)**
```bash
# Solution: Check permissions on ~/.ssh
chmod 700 ~/.ssh
chmod 600 ~/.ssh/id_rsa
chmod 644 ~/.ssh/id_rsa.pub

# Check server authorized_keys
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys
```

**Issue: Connection Refused**
```bash
# Solution: Check if SSH service is running
sudo systemctl status ssh
sudo systemctl start ssh

# Check SSH listening on port 22
sudo netstat -tlnp | grep ssh
```

**Issue: Host key verification failed**
```bash
# Solution: Add host to known_hosts
ssh-keyscan hostname >> ~/.ssh/known_hosts
# or use
ssh -o StrictHostKeyChecking=no user@hostname
```

**Issue: Too many authentication failures**
```bash
# Solution: Specify the correct key
ssh -i ~/.ssh/correct_key user@hostname

# Clear SSH agent
ssh-add -D
ssh-add ~/.ssh/correct_key
```

### Advanced Configuration
```bash
# Edit SSH config file
nano ~/.ssh/config

# Example config:
Host myserver
    HostName example.com
    User username
    IdentityFile ~/.ssh/id_rsa
    Port 22

# Use config
ssh myserver

# Debug connection issues
ssh -vvv user@hostname

# Generate different key types
ssh-keygen -t ecdsa -b 256 -f ~/.ssh/id_ecdsa
```

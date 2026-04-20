# SSH - Senior Level Interview Questions & Answers

## Table of Contents
1. Advanced Authentication
2. Security Hardening
3. Tunneling & Proxying
4. Scaling & Management
5. Auditing & Compliance

---

## 1. Advanced Authentication

### Q (High-Level): Why do we use SSH instead of just remembering passwords?

**A (Real-Time Security):**
Your app needs to connect to database server. How?

**Bad approach - Password:**
```bash
# Store password in config file
#!/bin/bash
DB_HOST=db.company.com
DB_USER=admin
DB_PASS=MyPassword123  # 😱 HUGE SECURITY RISK!

ssh $DB_USER@$DB_HOST
```

**Problems:**
- Anyone with file access sees password
- Password reuse across servers (one leak = all servers compromised)
- Can't revoke access instantly
- No audit trail of who accessed what

**Good approach - SSH Keys:**
```bash
# Use key-pair (private key on your machine, public key on server)
ssh -i ~/.ssh/id_rsa admin@db.company.com

# Advantages:
# - Key never transmitted (only used for authentication)
# - Instant access revocation (delete public key)
# - Different key per server (one leak = one server affected)
# - Can enforce key authentication only (no passwords)
```

**Real-time scenario:**
```
Employee leaves company Friday:
- With passwords: Remove from 50 servers manually, might miss one
- With SSH keys: Delete their public key from each server, access revoked immediately ✅
```

---

### Q (High-Level): Real-time incident - Hacker compromised a server and might have stolen SSH keys. What's your emergency response?

**A (Security Breach Crisis):**
```
Alert: "Server hacked, attacker in for 2 hours"
Hacker might have stolen: /home/user/.ssh/id_rsa (private key)
```

**Immediate actions:**
```bash
# 1. Check what keys are authorized on server
cat ~/.ssh/authorized_keys  # Shows all compromised keys

# 2. Find which servers the hacker could access
# If hacker has your key, they can SSH to ANY server with public key

# 3. REVOKE all your keys
# On every server: Remove YOUR public key from authorized_keys
# for i in {1..100}; do
#   ssh server-$i "sed -i '/your-username/d' ~/.ssh/authorized_keys"
# done

# 4. Generate NEW keys
ssh-keygen -t ed25519 -f ~/.ssh/id_rsa_new

# 5. Add new keys to all servers
# (This takes an hour, but necessary)
```

**Prevention for next time:**
```bash
# Use certificate-based auth instead of keys
# Certificates have expiration date (e.g., 24 hours)
# Even if hacked, certificate expires automatically
```

**Real-time impact:**
- Without quick response: Hacker accesses all 100 servers
- With quick response (1 hour): Hacker locked out

---

### Q: Design a zero-trust SSH authentication architecture

**A:**
```bash
# Zero-Trust SSH Architecture

# 1. Certificate-based authentication (not passwords)
# Generate CA certificate
ssh-keygen -t ed25519 -f ssh_host_ca_key -N ""
ssh-keygen -t ed25519 -f ssh_user_ca_key -N ""

# Create certificate
ssh-keygen -s ssh_user_ca_key \
  -I "user@karthik" \
  -n karthik \
  -V +52w \
  /home/karthik/.ssh/id_ed25519.pub

# Server configuration
cat >> /etc/ssh/sshd_config << 'EOF'
# Use certificate for user authentication
HostKey /etc/ssh/ssh_host_ed25519_key
HostCertificate /etc/ssh/ssh_host_ed25519_key-cert.pub

# Trust CA certificate
TrustedUserCAKeys /etc/ssh/ssh_user_ca_key.pub

# Disable password auth
PasswordAuthentication no
PubkeyAuthentication yes

# Key-only methods
AuthenticationMethods publickey
EOF

# 2. PIV/FIDO2 hardware keys
# Requires OpenSSH 8.2+

# List available keys
ssh-add -l

# Add FIDO2 key
ssh-add -t ed25519-sk

# SSH with FIDO2
ssh -l karthik bastion.example.com

# 3. Multi-factor authentication
cat >> /etc/ssh/sshd_config << 'EOF'
# Require both certificate and password
AuthenticationMethods publickey,keyboard-interactive
EOF

# 4. Certificate with restrictions
ssh-keygen -s ssh_user_ca_key \
  -I "user@karthik" \
  -n karthik \
  -O no-agent-forwarding \
  -O no-port-forwarding \
  -O no-pty \
  -O no-user-rc \
  -O force-command="echo 'SSH Certificate Active'" \
  -V +1h \
  /home/karthik/.ssh/id_ed25519.pub

# 5. OpenSSH CA principal validation
# Add principals to identify access levels
ssh-keygen -s ssh_user_ca_key \
  -I "user@karthik" \
  -n karthik,admin,developer \
  /home/karthik/.ssh/id_ed25519.pub
```

---

### Q: Implement SSH key rotation and certificate management at scale

**A:**
```bash
#!/bin/bash
# Enterprise SSH certificate management system

# Configuration
CA_KEY="/etc/ssh/ca/ssh_user_ca_key"
CERT_VALIDITY="+52w"
CERT_PATH="/var/lib/ssh-certs"
AUDIT_LOG="/var/log/ssh-cert-audit.log"

# Certificate lifecycle manager
manage_certificates() {
    local user="$1"
    local action="$2"
    
    case "$action" in
        issue)
            issue_certificate "$user"
            ;;
        revoke)
            revoke_certificate "$user"
            ;;
        rotate)
            rotate_certificate "$user"
            ;;
        list)
            list_certificates "$user"
            ;;
    esac
}

# Issue new certificate
issue_certificate() {
    local user="$1"
    local key_path="$HOME/.ssh/id_ed25519.pub"
    
    if [[ ! -f "$key_path" ]]; then
        echo "ERROR: Public key not found for $user"
        return 1
    fi
    
    # Generate certificate
    ssh-keygen -s "$CA_KEY" \
        -I "$user@$(hostname)" \
        -n "$user" \
        -O no-agent-forwarding \
        -O no-port-forwarding \
        -V "$CERT_VALIDITY" \
        "$key_path"
    
    # Log action
    echo "$(date): Certificate issued for $user" >> "$AUDIT_LOG"
}

# Rotate certificates before expiration
rotate_certificate() {
    local user="$1"
    
    # Check if certificate expires in 30 days
    local expiry=$(ssh-keygen -L -f "${CERT_PATH}/${user}-cert.pub" | grep "Valid" | awk '{print $NF}')
    local days_until_expiry=$(( ($(date -d "$expiry" +%s) - $(date +%s)) / 86400 ))
    
    if [[ $days_until_expiry -lt 30 ]]; then
        echo "Rotating certificate for $user (expires in $days_until_expiry days)"
        issue_certificate "$user"
    fi
}

# Revoke certificate
revoke_certificate() {
    local user="$1"
    
    # Add to revocation list
    echo "$(date +%s):$(ssh-keygen -L -f ${CERT_PATH}/${user}-cert.pub | grep Serial)" >> /etc/ssh/revoked-keys
    
    # Reload SSHD
    systemctl reload sshd
    
    echo "$(date): Certificate revoked for $user" >> "$AUDIT_LOG"
}

# List active certificates
list_certificates() {
    local user="$1"
    
    for cert in ${CERT_PATH}/${user}*-cert.pub; do
        if [[ -f "$cert" ]]; then
            echo "Certificate: $cert"
            ssh-keygen -L -f "$cert"
        fi
    done
}

# Automated rotation job (cron)
rotate_all_certificates() {
    for user in /home/*/; do
        username=$(basename "$user")
        rotate_certificate "$username"
    done
}

# Usage
manage_certificates "karthik" "issue"
manage_certificates "karthik" "list"
manage_certificates "karthik" "rotate"
```

---

## 2. Security Hardening

### Q: Design hardened SSH configuration for production

**A:**
```bash
# /etc/ssh/sshd_config - Hardened configuration

# Port (non-standard)
Port 2222

# Protocol version
Protocol 2

# HostKey configuration
HostKey /etc/ssh/ssh_host_ed25519_key
HostKey /etc/ssh/ssh_host_rsa_key

# Cipher configuration (strong only)
Ciphers chacha20-poly1305@openssh.com,aes-256-gcm@openssh.com,aes-128-gcm@openssh.com
MACs hmac-sha2-512-etm@openssh.com,hmac-sha2-256-etm@openssh.com
KexAlgorithms curve25519-sha256,curve25519-sha256@libssh.org
HostKeyAlgorithms ssh-ed25519

# Authentication
PasswordAuthentication no
PubkeyAuthentication yes
PermitEmptyPasswords no
MaxAuthTries 3
MaxSessions 2

# Use certificates
HostCertificate /etc/ssh/ssh_host_ed25519_key-cert.pub
TrustedUserCAKeys /etc/ssh/ssh_user_ca_key.pub

# Disable dangerous features
PermitRootLogin no
PermitUserEnvironment no
AllowAgentForwarding no
AllowTcpForwarding no
PermitTunnel no
X11Forwarding no

# Restrict subsystems
Subsystem sftp internal-sftp -f AUTHPRIV -l VERBOSE

# Timeouts
ClientAliveInterval 300
ClientAliveCountMax 2
LoginGraceTime 30

# Logging
SyslogFacility AUTH
LogLevel VERBOSE

# Access control
AllowUsers karthik@10.0.0.* karthik@192.168.*

# Banner
Banner /etc/ssh/banner.txt

# Restart SSHD
systemctl restart sshd
systemctl enable sshd
```

---

### Q: Implement SSH bastion/jump host with audit trail

**A:**
```bash
# Bastion host configuration with full audit

# /etc/ssh/sshd_config on bastion
# Enable all features
AllowAgentForwarding yes
AllowTcpForwarding yes
PermitTTY yes

# Force command wrapper for auditing
ForceCommand /usr/local/bin/ssh-audit-wrapper.sh

# Create audit wrapper
cat > /usr/local/bin/ssh-audit-wrapper.sh << 'EOF'
#!/bin/bash

# SSH Audit Wrapper
AUDIT_LOG="/var/log/ssh-audit/bastion.log"
RECORDING_DIR="/var/log/ssh-audit/sessions"

# Log connection details
{
    echo "$(date '+%Y-%m-%d %H:%M:%S')"
    echo "User: $SSH_ORIGINAL_COMMAND"
    echo "From: $SSH_CLIENT"
    echo "Session: $SSH_SESSION_ID"
    echo "---"
} >> "$AUDIT_LOG"

# Record session (using asciinema or script)
mkdir -p "$RECORDING_DIR"
SESSION_FILE="$RECORDING_DIR/$(date +%Y%m%d_%H%M%S)_${REMOTE_USER}.cast"

script -q -t 2>"${SESSION_FILE}.timing" -f "${SESSION_FILE}.log" \
    /bin/bash -c "$SSH_ORIGINAL_COMMAND"

# Send alert if root activity
if [[ "$SSH_ORIGINAL_COMMAND" =~ sudo|su|root ]]; then
    echo "ALERT: Privilege escalation attempted by $REMOTE_USER" | \
        mail -s "SSH Alert" security@example.com
fi
EOF

chmod 755 /usr/local/bin/ssh-audit-wrapper.sh

# ProxyJump client configuration
cat > ~/.ssh/config << 'EOF'
# Bastion configuration
Host bastion
    HostName bastion.example.com
    Port 2222
    User karthik
    IdentityFile ~/.ssh/id_ed25519
    ControlMaster auto
    ControlPath ~/.ssh/control-%h-%p-%r

# Jump through bastion
Host app-server
    HostName app.internal.example.com
    ProxyJump bastion
    User karthik
    IdentityFile ~/.ssh/id_ed25519

# Connect
ssh app-server
EOF

# Session recording playback
cat > /usr/local/bin/ssh-playback.sh << 'EOF'
#!/bin/bash
# Playback recorded session

SESSION_FILE="$1"
if [[ -f "${SESSION_FILE}.timing" ]]; then
    scriptreplay -t "${SESSION_FILE}.timing" -s 0.1 "${SESSION_FILE}.log"
fi
EOF

chmod 755 /usr/local/bin/ssh-playback.sh
```

---

## 3. Tunneling & Proxying

### Q: Design SSH tunneling strategy for secure connectivity

**A:**
```bash
# SSH Tunneling Patterns

# 1. Local port forwarding (access internal service)
ssh -L 8080:internal-database:5432 bastion.example.com
# Now: localhost:8080 -> bastion -> internal-database:5432

# 2. Remote port forwarding (expose local service)
ssh -R 8080:localhost:3000 bastion.example.com
# Now: bastion:8080 -> local machine:3000

# 3. Dynamic SOCKS proxy
ssh -D 1080 bastion.example.com
# Configure browser proxy: socks5://localhost:1080

# 4. ProxyCommand for bastion
ssh -o ProxyCommand="ssh -W %h:%p bastion.example.com" target.internal.com

# 5. VPN-like setup (full tunnel)
ssh -L 0.0.0.0:8080:internal-service:8080 bastion.example.com

# 6. Multiple jump hosts
ssh -J bastion1.example.com,bastion2.example.com target.internal.com

# 7. Persistent tunnel script
#!/bin/bash
create_persistent_tunnel() {
    local tunnel_name="$1"
    local local_port="$2"
    local remote_host="$3"
    local remote_port="$4"
    local bastion="$5"
    
    # Check if tunnel exists
    if pgrep -f "ssh.*-L.*${local_port}"; then
        echo "Tunnel already active"
        return 0
    fi
    
    # Create tunnel in background
    nohup ssh -N -L ${local_port}:${remote_host}:${remote_port} ${bastion} \
        > /tmp/${tunnel_name}.log 2>&1 &
    
    echo "Tunnel created with PID $!"
}

# Usage
create_persistent_tunnel "db-tunnel" "5432" "internal-db" "5432" "bastion.example.com"
```

---

## 4. Scaling & Management

### Q: Design SSH key management for thousands of servers

**A:**
```bash
# Enterprise SSH key management

# 1. Key rotation automation
#!/bin/bash
rotate_all_keys() {
    local servers=$(cat /etc/hosts.inventory)
    local new_key="/tmp/new_key"
    
    # Generate new key
    ssh-keygen -t ed25519 -f "$new_key" -N ""
    
    # Deploy to all servers
    for server in $servers; do
        echo "Rotating keys on $server..."
        ssh "$server" "cat >> ~/.ssh/authorized_keys" < "${new_key}.pub"
        
        # Wait for verification
        sleep 5
        
        # Remove old keys
        ssh "$server" "sed -i '/old_key/d' ~/.ssh/authorized_keys"
    done
}

# 2. Centralized key distribution with Vault
vault write ssh/roles/my-role \
  key_type=ca \
  ttl=30m \
  max_ttl=2h \
  allowed_users="karthik,deploy,admin"

# Request certificate
vault read -field=signed_key ssh/sign/my-role public_key=@~/.ssh/id_ed25519.pub

# 3. Ansible-based deployment
---
- hosts: all
  tasks:
    - name: Deploy SSH keys
      authorized_key:
        user: karthik
        state: present
        key: "{{ lookup('file', 'files/id_ed25519.pub') }}"
```

---

## 5. Auditing & Compliance

### Q: Design SSH auditing for compliance (SOC2, HIPAA, PCI-DSS)

**A:**
```bash
# SSH Auditing Configuration

# 1. Enhanced logging
cat >> /etc/ssh/sshd_config << 'EOF'
# Detailed logging
LogLevel VERBOSE
SyslogFacility AUTH

# Log all successful connections
LogLevel DEBUG3

# Specify log file
SyslogFacility AUTH
LogLevel VERBOSE
EOF

# 2. Audit script
#!/bin/bash
# Monitor SSH access

log_ssh_access() {
    # Extract failed attempts
    grep "Failed password" /var/log/auth.log | \
        awk '{print $1, $2, $3, $9}' > /var/log/ssh-failed.log
    
    # Extract successful logins
    grep "Accepted publickey" /var/log/auth.log | \
        awk '{print $1, $2, $3, $9, $11, $13}' > /var/log/ssh-accepted.log
    
    # Alert on suspicious activity
    failed_count=$(wc -l < /var/log/ssh-failed.log)
    if [[ $failed_count -gt 5 ]]; then
        echo "Alert: $failed_count failed login attempts" | \
            mail -s "SSH Security Alert" security@example.com
    fi
}

# 3. Compliance reporting
generate_ssh_report() {
    {
        echo "=== SSH Security Report ==="
        echo "Date: $(date)"
        echo ""
        echo "=== SSH Configuration Check ==="
        
        # Check for weak ciphers
        grep -i cipher /etc/ssh/sshd_config
        
        echo ""
        echo "=== Failed Login Attempts (Last 24h) ==="
        grep "Failed password" /var/log/auth.log | \
            grep "$(date '+%b %d')" | wc -l
        
        echo ""
        echo "=== Successful Connections (Last 24h) ==="
        grep "Accepted publickey" /var/log/auth.log | \
            grep "$(date '+%b %d')" | wc -l
        
    } > /var/log/ssh-compliance-report-$(date +%Y%m%d).txt
}

cron entry for daily reports:
0 2 * * * /usr/local/bin/generate_ssh_report.sh
```

---

## Summary

**Critical Senior SSH Skills:**
1. ✅ Certificate-based authentication
2. ✅ Zero-trust architecture
3. ✅ Hardened configurations
4. ✅ Bastion/jump host setup
5. ✅ SSH tunneling patterns
6. ✅ Key management at scale
7. ✅ Auditing & compliance
8. ✅ Session recording

**Created by:** Karthik Reddy Vaddepal

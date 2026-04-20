# Authentication - Authentication Mechanisms & Services

## What is it used for?
Authentication is used for:
- **User verification**: Verify user identity
- **Access control**: Control who can access resources
- **Single Sign-On (SSO)**: Centralized authentication
- **Multi-factor authentication**: Add security layers
- **OAuth/OpenID Connect**: Delegated authentication
- **LDAP/Active Directory**: Enterprise authentication
- **API authentication**: Authenticate API requests
- **Token management**: JWT, refresh tokens

## Installation & Setup

```bash
# LDAP Server (OpenLDAP)
sudo apt-get install slapd ldap-utils

# PAM (Pluggable Authentication Modules)
sudo apt-get install libpam-dev

# OpenID Connect libraries
pip install python-jose
pip install flask-oidc

# JWT libraries
pip install pyjwt

# OAuth2 proxy
wget https://github.com/oauth2-proxy/oauth2-proxy/releases/download/v7.4.0/oauth2-proxy-v7.4.0.linux-amd64.go18.tar.gz
```

## Basic Commands & Troubleshooting

### PAM Configuration
```bash
# View PAM configuration
cat /etc/pam.d/common-auth

# Enable 2FA with TOTP
sudo apt-get install libpam-google-authenticator

# Configure TOTP
google-authenticator

# Edit PAM config
sudo nano /etc/pam.d/sshd
# Add: auth required pam_google_authenticator.so
```

### LDAP Authentication
```bash
# Test LDAP connection
ldapwhoami -h ldap.example.com -D "cn=admin,dc=example,dc=com" -W

# Search LDAP directory
ldapsearch -h ldap.example.com -D "cn=admin,dc=example,dc=com" \
  -W -b "dc=example,dc=com" "uid=username"

# Add user to LDAP
ldapadd -h ldap.example.com -D "cn=admin,dc=example,dc=com" -W -f user.ldif

# Modify LDAP entry
ldapmodify -h ldap.example.com -D "cn=admin,dc=example,dc=com" -W < modify.ldif

# Delete LDAP entry
ldapdelete -h ldap.example.com -D "cn=admin,dc=example,dc=com" \
  -W "uid=username,ou=people,dc=example,dc=com"
```

### JWT Token Management
```bash
# Python JWT creation
python3 << 'EOF'
import jwt
import json
from datetime import datetime, timedelta

payload = {
    'user_id': 123,
    'exp': datetime.utcnow() + timedelta(hours=1)
}
token = jwt.encode(payload, 'secret', algorithm='HS256')
print(token)
EOF

# Verify JWT token
python3 << 'EOF'
import jwt
token = "your_token_here"
decoded = jwt.decode(token, 'secret', algorithms=['HS256'])
print(decoded)
EOF
```

### OAuth2 Configuration
```bash
# OAuth2 Proxy configuration
cat > /etc/oauth2-proxy/oauth2-proxy.cfg << 'EOF'
upstream http_backend {
  server 127.0.0.1:8080;
}

auth_basic "OAuth2 Protected";
auth_basic_user_file /etc/nginx/.htpasswd;

location / {
  auth_request /oauth2/auth;
  error_page 401 /oauth2/sign_in;
  proxy_pass http_backend;
}

location = /oauth2/auth {
  proxy_pass http://127.0.0.1:4180;
}

location /oauth2/ {
  proxy_pass http://127.0.0.1:4180;
}
EOF

# Start OAuth2 Proxy
oauth2-proxy --config=/etc/oauth2-proxy/oauth2-proxy.cfg
```

### OpenID Connect
```bash
# Python OIDC client
python3 << 'EOF'
from flask_oidc import OpenIDConnect
from flask import Flask

app = Flask(__name__)
oidc = OpenIDConnect(app)

@app.route('/')
@oidc.require_login
def index():
    return 'Hello %s' % oidc.user_getfield('email')

if __name__ == '__main__':
    app.run()
EOF
```

### Active Directory
```bash
# Configure Linux to use AD
sudo apt-get install libnss-ldap libpam-ldap

# Edit /etc/ldap.conf
sudo nano /etc/ldap.conf
# host ad.example.com
# base dc=example,dc=com

# Test AD connection
sudo getent passwd username@example.com

# Join domain
sudo realm join -U administrator example.com
```

### SSH Key Authentication
```bash
# Generate SSH key
ssh-keygen -t ed25519 -C "user@example.com"

# Copy SSH key to server
ssh-copy-id -i ~/.ssh/id_ed25519.pub user@hostname

# SSH with specific key
ssh -i ~/.ssh/id_ed25519 user@hostname

# SSH config for key
cat >> ~/.ssh/config << 'EOF'
Host myserver
  HostName example.com
  User username
  IdentityFile ~/.ssh/id_ed25519
EOF
```

### Multi-Factor Authentication (MFA)
```bash
# Enable SSH key + password
sudo nano /etc/ssh/sshd_config
# AuthenticationMethods publickey,password

# Enable SSH key + OTP
# AuthenticationMethods publickey,keyboard-interactive

# Verify MFA setup
sudo sshd -T | grep -i authentication

# Configure U2F/FIDO2
sudo apt-get install libpam-u2f

# Register FIDO2 key
pamu2fcfg > ~/.config/Yubico/u2f_keys
```

### Common Issues & Resolution

**Issue: LDAP authentication fails**
```bash
# Solution: Test LDAP connectivity
ldapwhoami -h ldap.example.com -D "cn=admin,dc=example,dc=com" -W

# Check LDAP logs
sudo tail -f /var/log/syslog | grep ldap

# Verify LDAP configuration
ldapcert info -h ldap.example.com

# Test with verbose output
ldapsearch -v -h ldap.example.com -D "cn=admin,dc=example,dc=com" -W
```

**Issue: JWT token expired**
```bash
# Solution: Refresh token
python3 << 'EOF'
import jwt
from datetime import datetime, timedelta

old_token = "your_expired_token"
decoded = jwt.decode(old_token, options={"verify_signature": False})

# Create new token
new_payload = {**decoded, 'exp': datetime.utcnow() + timedelta(hours=1)}
new_token = jwt.encode(new_payload, 'secret', algorithm='HS256')
print(new_token)
EOF
```

**Issue: OAuth2 redirect URI mismatch**
```bash
# Solution: Check OAuth2 application settings
# Verify redirect URI matches:
# https://your-app.com/oauth2/callback

# Update if needed in OAuth provider settings
```

### Debugging
```bash
# Test authentication
ssh -v user@hostname  # Verbose SSH

# Check PAM logs
sudo grep PAM /var/log/auth.log

# LDAP debug
ldapwhoami -v -h ldap.example.com

# JWT decode (online or CLI)
echo "eyJ..." | cut -d'.' -f2 | base64 -d | jq

# OAuth2 proxy logs
tail -f /var/log/oauth2-proxy.log

# Check active sessions
w  # Who's logged in
```

### Security Best Practices
```bash
# Use strong password policy
# /etc/security/pwquality.conf
minlen = 14
dcredit = -1
ucredit = -1
ocredit = -1
lcredit = -1

# Enforce account lockout
# /etc/pam.d/common-auth
pam_tally2.so onerr=fail audit silent deny=5 unlock_time=900

# Session timeout
session required pam_limits.so

# Force password change
chage -d 0 username
```

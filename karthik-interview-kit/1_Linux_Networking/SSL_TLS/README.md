# SSL/TLS - Secure Sockets Layer / Transport Layer Security

## What is it used for?
SSL/TLS protocols provide:
- **Encryption**: Encrypt data in transit between client and server
- **Authentication**: Verify identity of servers and clients
- **Data integrity**: Ensure data hasn't been tampered with
- **HTTPS**: Secure web communications
- **Secure email**: Encrypted email protocols
- **VPN tunnels**: Encrypted network tunnels
- **API security**: Secure REST API communications

## Installation

```bash
# OpenSSL (most common)
apt-get install openssl        # Debian/Ubuntu
yum install openssl            # RHEL/CentOS

# Certificate management tools
apt-get install certbot        # Let's Encrypt client
```

## Basic Commands & Troubleshooting

### Certificate Operations
```bash
# Generate private key
openssl genrsa -out private.key 2048

# Generate CSR (Certificate Signing Request)
openssl req -new -key private.key -out certificate.csr

# Generate self-signed certificate
openssl req -x509 -newkey rsa:2048 -keyout key.pem -out cert.pem -days 365

# View certificate details
openssl x509 -in certificate.crt -text -noout

# View CSR details
openssl req -in certificate.csr -text -noout

# View key information
openssl rsa -in private.key -text -noout

# Check expiration date
openssl x509 -in certificate.crt -noout -dates

# Convert certificate formats
openssl x509 -in cert.pem -outform der -out cert.der  # PEM to DER
openssl x509 -in cert.der -inform der -outform pem -out cert.pem  # DER to PEM

# Verify certificate
openssl verify certificate.crt

# Check certificate chain
openssl verify -CAfile ca.crt certificate.crt
```

### Testing SSL/TLS Connection
```bash
# Check SSL certificate of a website
openssl s_client -connect example.com:443

# Verbose connection test
openssl s_client -connect example.com:443 -showcerts

# Test specific TLS version
openssl s_client -connect example.com:443 -tls1_2

# Check certificate expiration
echo | openssl s_client -connect example.com:443 | openssl x509 -noout -dates
```

### Common Issues & Resolution

**Issue: Certificate has expired**
```bash
# Check expiration
openssl x509 -in certificate.crt -noout -dates

# Solution: Renew certificate
openssl req -new -key private.key -out certificate.csr
# Send CSR to CA for renewal
```

**Issue: Self-signed certificate warning**
```bash
# Create self-signed certificate
openssl req -x509 -newkey rsa:2048 \
  -keyout key.pem -out cert.pem -days 365

# For client to trust, import certificate to trusted store
# Or use: -k flag with curl to skip verification (not recommended)
curl -k https://example.com
```

**Issue: Certificate mismatch**
```bash
# Check CN (Common Name) in certificate
openssl x509 -in certificate.crt -noout -text | grep -A1 "Subject:"

# Issue: CN doesn't match hostname
# Solution: Obtain correct certificate or update hostname
```

**Issue: Private key and certificate don't match**
```bash
# Compare modulus values
openssl rsa -in private.key -noout -modulus | openssl md5
openssl x509 -in certificate.crt -noout -modulus | openssl md5

# If different, they don't match. Get correct pair.
```

### Using Let's Encrypt (Free SSL)
```bash
# Install Certbot
apt-get install certbot python3-certbot-nginx

# Generate certificate
sudo certbot certonly --standalone -d example.com

# Automatic renewal
sudo certbot renew --dry-run

# View certificates
sudo certbot certificates
```

### Server Configuration
```bash
# Enable SSL in Nginx
# /etc/nginx/sites-available/example.com
server {
    listen 443 ssl http2;
    server_name example.com;
    ssl_certificate /etc/ssl/certs/certificate.crt;
    ssl_certificate_key /etc/ssl/private/private.key;
}

# Enable SSL in Apache
# /etc/apache2/sites-available/example.com.conf
<VirtualHost *:443>
    SSLEngine on
    SSLCertificateFile /etc/ssl/certs/certificate.crt
    SSLCertificateKeyFile /etc/ssl/private/private.key
</VirtualHost>
```

### Troubleshooting Commands
```bash
# Check which certificates are loaded by a service
sudo ss -tlnp | grep ssl

# Monitor certificate expiration
for cert in /etc/ssl/certs/*.crt; do
  echo "Certificate: $cert"
  openssl x509 -in "$cert" -noout -dates
done

# Test TLS connection strength
nmap --script ssl-enum-ciphers -p 443 example.com
```

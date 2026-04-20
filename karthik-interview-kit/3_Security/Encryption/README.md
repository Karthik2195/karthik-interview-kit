# Encryption - Data Encryption Methods & Tools

## What is it used for?
Encryption is used for:
- **Data protection**: Encrypt sensitive data at rest
- **Communication security**: Encrypt data in transit
- **Compliance**: Meet regulatory requirements (HIPAA, GDPR, PCI-DSS)
- **Key management**: Securely store and manage encryption keys
- **Certificate management**: SSL/TLS certificates
- **Application security**: Protect application data
- **Database encryption**: Encrypt database contents
- **Disk encryption**: Full disk encryption

## Installation & Setup

```bash
# OpenSSL
sudo apt-get install openssl

# GPG (GNU Privacy Guard)
sudo apt-get install gnupg

# LibreSSL
sudo apt-get install libressl

# Cryptsetup (disk encryption)
sudo apt-get install cryptsetup

# HashiCorp Vault
wget https://releases.hashicorp.com/vault/1.15.0/vault_1.15.0_linux_amd64.zip
unzip vault_1.15.0_linux_amd64.zip
sudo mv vault /usr/local/bin/
```

## Basic Commands & Troubleshooting

### Symmetric Encryption
```bash
# Encrypt file with OpenSSL (AES-256)
openssl enc -aes-256-cbc -salt -in file.txt -out file.txt.enc -k password

# Decrypt file
openssl enc -d -aes-256-cbc -in file.txt.enc -out file.txt -k password

# Encrypt with passphrase prompt
openssl enc -aes-256-cbc -salt -in file.txt -out file.txt.enc

# Decrypt with prompt
openssl enc -d -aes-256-cbc -in file.txt.enc -out file.txt
```

### Asymmetric Encryption (RSA)
```bash
# Generate RSA key pair
openssl genrsa -out private.key 2048
openssl rsa -in private.key -pubout -out public.key

# Encrypt with public key
openssl rsautl -encrypt -inkey public.key -pubin -in file.txt -out file.txt.enc

# Decrypt with private key
openssl rsautl -decrypt -inkey private.key -in file.txt.enc -out file.txt
```

### GPG Encryption
```bash
# Generate GPG key pair
gpg --gen-key

# List keys
gpg --list-keys

# Export public key
gpg --export -a "user@example.com" > public.key

# Encrypt file with GPG
gpg --encrypt --recipient "user@example.com" file.txt

# Decrypt file
gpg --decrypt file.txt.gpg > file.txt

# Sign document
gpg --sign --local-user "user@example.com" file.txt

# Verify signature
gpg --verify file.txt.sig
```

### LUKS Disk Encryption
```bash
# Create encrypted partition
sudo cryptsetup luksFormat /dev/sdb1

# Open encrypted partition
sudo cryptsetup luksOpen /dev/sdb1 my-encrypted-disk

# Format encrypted partition
sudo mkfs.ext4 /dev/mapper/my-encrypted-disk

# Mount encrypted partition
sudo mount /dev/mapper/my-encrypted-disk /mnt/encrypted

# Close encrypted partition
sudo cryptsetup luksClose my-encrypted-disk

# Check LUKS header
sudo cryptsetup luksDump /dev/sdb1
```

### Database Encryption

**PostgreSQL Encryption**
```bash
# Install pgcrypto extension
psql -c "CREATE EXTENSION pgcrypto;"

# Encrypt data
SELECT pgp_sym_encrypt('secret data', 'password');

# Decrypt data
SELECT pgp_sym_decrypt(encrypted_data, 'password');

# Hash password
SELECT crypt('password', gen_salt('bf'));
```

**MySQL Encryption**
```bash
# Enable key ring plugin
# In /etc/mysql/mysql.conf.d/mysqld.cnf
# Add: plugin-load-add=keyring_file.so

# Encrypt table
ALTER TABLE users ENCRYPTION='Y';

# Check encryption status
SELECT table_schema, table_name, table_encryption 
FROM information_schema.tables 
WHERE table_encryption='Y';
```

### HashiCorp Vault
```bash
# Start Vault server (dev mode)
vault server -dev

# Set VAULT_ADDR
export VAULT_ADDR='http://127.0.0.1:8200'

# List secrets
vault secrets list

# Store secret
vault kv put secret/myapp/database password="mypassword"

# Retrieve secret
vault kv get secret/myapp/database

# Delete secret
vault kv delete secret/myapp/database

# Encrypt data
vault write transit/encrypt/mykey plaintext=@file.txt

# Decrypt data
vault write transit/decrypt/mykey ciphertext=...
```

### Common Issues & Resolution

**Issue: Wrong password for encryption**
```bash
# Solution: Check encryption parameters
openssl enc -aes-256-cbc -d -in file.enc -out file.txt -k password

# Try different algorithm if unsure
# List available algorithms
openssl enc -ciphers
```

**Issue: LUKS partition locked**
```bash
# Solution: Open partition with correct passphrase
sudo cryptsetup luksOpen /dev/sdb1 my-disk

# Check partition status
sudo cryptsetup status /dev/mapper/my-disk

# Force close if needed
sudo cryptsetup luksClose /dev/mapper/my-disk
```

**Issue: Cannot read encrypted database**
```bash
# Solution: Verify encryption key availability
# Check key management service (KMS)

# For PostgreSQL:
SELECT * FROM pg_stat_ssl WHERE pid = pg_backend_pid();

# For MySQL:
SHOW VARIABLES LIKE 'keyring%';
```

### Debugging
```bash
# Test OpenSSL encryption
echo "test data" | openssl enc -aes-256-cbc -salt -P -k password

# Verify encrypted file
file file.txt.enc

# Check GPG key trust
gpg --check-trustdb

# Display encryption details
openssl enc -aes-256-cbc -P -in file.txt

# Vault audit log
vault audit list

# Monitor encryption operations
sudo strace -e openat,read,write openssl enc ...
```

### Key Rotation
```bash
# Rotate database encryption keys
# Oracle:
ALTER SYSTEM SET db_recovery_file_dest_size=...

# PostgreSQL:
ALTER SYSTEM SET ssl_key_file = 'new_key.key';

# MySQL:
ALTER INSTANCE ROTATE INNODB_MASTER_KEY;

# Check rotated keys
SHOW BINARY LOGS;
```

### Security Best Practices
```bash
# Use strong algorithms
openssl enc -aes-256-cbc  # Good

# Avoid weak algorithms
openssl enc -des-cbc  # Deprecated

# Use proper key derivation
openssl enc -aes-256-cbc -pbkdf2

# Securely store keys
chmod 600 private.key

# Never commit keys to git
echo "private.key" >> .gitignore
```

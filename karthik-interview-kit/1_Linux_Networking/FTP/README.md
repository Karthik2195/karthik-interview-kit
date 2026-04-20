# FTP - File Transfer Protocol

## What is it used for?
FTP (File Transfer Protocol) is used for:
- **File transfer**: Upload and download files between computers
- **Legacy systems**: Connect to older systems that use FTP
- **Server management**: Access remote file systems
- **Backup operations**: Automate file backups
- **Public file sharing**: Share files publicly
- **Data migration**: Transfer large amounts of data

## Installation

```bash
# Linux - FTP Server
apt-get install vsftpd      # Debian/Ubuntu
yum install vsftpd          # RHEL/CentOS
systemctl start vsftpd      # Start service

# Linux - FTP Client
apt-get install ftp         # Debian/Ubuntu
yum install ftp             # RHEL/CentOS
```

## Basic Commands & Troubleshooting

### Essential Commands
```bash
# Connect to FTP server
ftp hostname
ftp ftp.example.com

# Connect with username
ftp -l username hostname

# Upload file
put filename.txt

# Download file
get filename.txt

# List files
ls
dir

# Change directory
cd /path/

# Create directory
mkdir directoryname

# Delete file
delete filename.txt

# Remove directory
rmdir directoryname

# Rename file
rename oldname.txt newname.txt

# Exit FTP
quit
bye

# Transfer binary files
binary

# Transfer text files
ascii

# Passive mode
passive

# Active mode
active
```

### Using LFTP (Advanced FTP Client)
```bash
# Install LFTP
apt-get install lftp

# Connect to FTP server
lftp username@hostname

# Recursive upload
mirror -R /local/path /remote/path

# Recursive download
mirror /remote/path /local/path

# Execute commands
lftp -u username,password hostname
```

### Automated FTP with Scripts
```bash
# Script for batch upload
ftp -n << 'EOF'
open hostname
user username password
binary
cd /remote/path
put local_file.txt
quit
EOF

# Script for batch download
ftp -n << 'EOF'
open hostname
user username password
binary
cd /remote/path
get remote_file.txt
quit
EOF
```

### Common Issues & Resolution

**Issue: Connection refused**
```bash
# Solution: Check FTP service
sudo systemctl status vsftpd
sudo systemctl restart vsftpd

# Check port 21
sudo netstat -tlnp | grep :21
```

**Issue: Permission denied**
```bash
# Solution: Check user permissions
sudo nano /etc/vsftpd.conf
# Ensure user is in allowed_users list

# Restart service
sudo systemctl restart vsftpd
```

**Issue: Passive mode issues**
```bash
# Solution: Configure passive mode range
sudo nano /etc/vsftpd.conf
# Add:
# pasv_min_port=6000
# pasv_max_port=6100

sudo systemctl restart vsftpd
```

**Issue: Cannot upload files**
```bash
# Solution: Check directory permissions
chmod 755 /var/ftp/upload
chown ftp:ftp /var/ftp/upload

# Ensure FTP user has write access
```

### Secure FTP Alternatives

**Use SFTP instead (Secure FTP over SSH)**
```bash
# Connect via SFTP
sftp user@hostname

# Similar commands as FTP
put filename.txt
get filename.txt
ls
exit
```

**Using scp for single file transfer**
```bash
scp filename.txt user@hostname:/remote/path
scp user@hostname:/remote/file.txt ./local/path
```

# Bash - Terminal Bash Scripting

## What is it used for?
Bash is the Bourne Again Shell, a Unix shell and command language used for:
- **Scripting automation**: Execute multiple commands in sequence
- **System administration**: Automate repetitive tasks
- **DevOps workflows**: Build pipelines and automation scripts
- **File operations**: Managing files and directories
- **Process management**: Running and monitoring processes
- **Text processing**: Using tools like grep, awk, sed

## Installation
```bash
# Linux (usually pre-installed)
apt-get install bash  # Debian/Ubuntu
yum install bash      # RHEL/CentOS

# Windows (Git Bash)
# Download from: https://gitforwindows.org/
```

## Basic Commands & Troubleshooting

### Essential Commands
```bash
# Check Bash version
bash --version

# Create a simple script
cat > script.sh << 'EOF'
#!/bin/bash
echo "Hello World"
EOF

# Make script executable
chmod +x script.sh

# Run script
./script.sh

# List files
ls -la

# Change directory
cd /path/to/directory

# Check current directory
pwd

# Create directory
mkdir -p /path/to/folder

# Copy files
cp source.txt destination.txt

# Move/rename files
mv oldname.txt newname.txt

# Remove files
rm file.txt

# Remove directory
rmdir directory/
```

### Common Issues & Resolution

**Issue: Permission Denied**
```bash
# Solution: Add execute permission
chmod +x script.sh
```

**Issue: Script not found**
```bash
# Solution: Use absolute or relative path
./script.sh  # Current directory
/home/user/script.sh  # Absolute path
```

**Issue: Command not found**
```bash
# Solution: Check PATH
echo $PATH
# Add to PATH if needed
export PATH=$PATH:/usr/local/bin
```

**Issue: Syntax errors in script**
```bash
# Solution: Debug mode
bash -x script.sh
# or
set -x  # Inside script
```

### Advanced Debugging
```bash
# Run with debug output
bash -x script.sh

# Check syntax without running
bash -n script.sh

# Show expanded commands
set -v

# Exit on error
set -e
```

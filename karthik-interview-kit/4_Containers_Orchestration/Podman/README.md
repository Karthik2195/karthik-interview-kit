# Podman - Container Management Platform

## What is it used for?
Podman is used for:
- **Container management**: Build, run, and manage containers
- **Rootless containers**: Run containers without root privileges
- **Docker compatibility**: Similar commands to Docker
- **Systemd integration**: Run containers as systemd services
- **Pod management**: Group containers together as pods
- **Image building**: Build container images
- **Daemonless architecture**: No background daemon required
- **Security**: Enhanced security for container isolation

## Installation

```bash
# Ubuntu/Debian
sudo apt-get update
sudo apt-get install podman

# RHEL/CentOS
sudo yum install podman

# Fedora
sudo dnf install podman

# macOS
brew install podman

# Verify installation
podman --version

# Test Podman
podman run hello-world
```

## Basic Commands & Troubleshooting

### Essential Commands
```bash
# Check Podman version
podman --version

# List containers
podman ps
podman ps -a  # All containers

# List images
podman images

# Pull image
podman pull ubuntu:20.04

# Run container
podman run ubuntu:20.04

# Run interactive container
podman run -it ubuntu:20.04 /bin/bash

# Run container in background
podman run -d ubuntu:20.04

# Run with name
podman run --name my-container ubuntu:20.04

# Stop container
podman stop container-id

# Start container
podman start container-id

# Remove container
podman rm container-id

# View container logs
podman logs container-id

# Execute command in container
podman exec -it container-id /bin/bash

# Build image
podman build -t image-name:tag .

# Push image to registry
podman push image-name:tag

# Tag image
podman tag image-name username/image-name:tag

# Remove image
podman rmi image-name

# Inspect container
podman inspect container-id

# Create pod
podman pod create --name my-pod

# Run container in pod
podman run -d --pod my-pod image-name

# List pods
podman pod ls

# Remove pod
podman pod rm pod-name
```

### Podman Compose
```bash
# Install Podman Compose
sudo curl -o /usr/local/bin/podman-compose https://raw.githubusercontent.com/containers/podman-compose/devel/podman-compose
sudo chmod +x /usr/local/bin/podman-compose

# Create docker-compose.yml
cat > docker-compose.yml << 'EOF'
version: '3.8'
services:
  web:
    image: nginx:latest
    ports:
      - "80:80"
  db:
    image: mysql:8.0
    environment:
      MYSQL_ROOT_PASSWORD: password
EOF

# Start services
podman-compose up -d

# Stop services
podman-compose down

# View logs
podman-compose logs

# Build services
podman-compose build
```

### Rootless Containers
```bash
# Set up user namespace
sudo loginctl enable-linger username

# Run rootless container
podman run -it ubuntu:20.04 /bin/bash

# Check if running rootless
podman info | grep rootless

# Configure cgroup v2
# Edit /etc/default/grub
# Add: GRUB_CMDLINE_LINUX="systemd.unified_cgroup_hierarchy=1"
# sudo update-grub

# Enable user namespaces
echo "user.max_user_namespaces=32768" | sudo tee /etc/sysctl.d/50-user-namespaces.conf
sudo sysctl -p /etc/sysctl.d/50-user-namespaces.conf
```

### Systemd Integration
```bash
# Run container as systemd service
podman generate systemd --name my-container > /etc/systemd/system/my-container.service

# Enable and start service
sudo systemctl enable my-container.service
sudo systemctl start my-container.service

# View service status
sudo systemctl status my-container.service

# View service logs
sudo journalctl -u my-container.service

# User-level systemd service
podman generate systemd --name my-container > ~/.config/systemd/user/my-container.service
systemctl --user enable my-container.service
systemctl --user start my-container.service
```

### Common Issues & Resolution

**Issue: Permission denied**
```bash
# Solution: Set up rootless
podman system migrate

# Or use sudo
sudo podman run image-name

# Add user to podman group
sudo usermod -aG podman $USER
newgrp podman
```

**Issue: Cgroup v1 not supported**
```bash
# Solution: Use cgroup v2
# Edit /etc/default/grub
GRUB_CMDLINE_LINUX="systemd.unified_cgroup_hierarchy=1"
sudo update-grub
sudo reboot

# Verify cgroup version
podman info | grep cgroupVersion
```

**Issue: Image pull fails**
```bash
# Solution: Check registry configuration
podman info | grep registries

# Edit registry config
nano ~/.config/containers/registries.conf

# Test registry connectivity
podman search ubuntu --registry docker.io
```

**Issue: Port binding fails**
```bash
# Solution: Use different port
podman run -p 8081:80 nginx

# Check port availability
sudo ss -tlnp | grep :80

# Use higher port number
podman run -p 8080:80 nginx
```

### Debugging
```bash
# Check Podman version
podman version

# Get system info
podman info

# List all containers (including stopped)
podman ps -a

# Inspect image details
podman inspect image-name

# Check storage information
podman info --format='{{json .Store}}'

# View container processes
podman top container-id

# Check network configuration
podman network ls
podman network inspect network-name

# Get CPU and memory stats
podman stats

# Check logs with timestamps
podman logs -t container-id

# View last N log lines
podman logs --tail 50 container-id
```

### Pod Management
```bash
# Create pod with port mapping
podman pod create --name my-pod -p 8080:80

# Run containers in pod
podman run -d --pod my-pod nginx
podman run -d --pod my-pod mysql

# Get pod info
podman pod inspect my-pod

# List pod containers
podman pod ps

# Remove pod (removes all containers)
podman pod rm my-pod

# Stop pod (stops all containers)
podman pod stop my-pod

# Start pod (starts all containers)
podman pod start my-pod
```

### Container Networking
```bash
# Create custom network
podman network create my-network

# Run container on network
podman run -d --network my-network --name web nginx

# Run another container on same network
podman run -d --network my-network --name app myapp

# Containers can communicate by name
# From app container: curl http://web

# List networks
podman network ls

# Inspect network
podman network inspect my-network

# Remove network
podman network rm my-network

# Connect container to network
podman network connect my-network container-id

# Disconnect container from network
podman network disconnect my-network container-id
```

### Advanced Features
```bash
# Mount volume
podman run -v /host/path:/container/path image-name

# Named volume
podman volume create my-volume
podman run -v my-volume:/container/path image-name

# List volumes
podman volume ls

# Remove volume
podman volume rm my-volume

# Copy files from container
podman cp container-id:/path/file.txt ./

# Copy files to container
podman cp file.txt container-id:/path/

# Create checkpoint
podman container checkpoint container-id

# Restore from checkpoint
podman container restore container-id
```

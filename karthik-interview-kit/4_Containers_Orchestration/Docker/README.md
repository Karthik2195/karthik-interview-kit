# Docker - Container Platform

## What is it used for?
Docker is used for:
- **Containerization**: Package applications with dependencies
- **Consistency**: Same environment across dev, test, and production
- **Microservices**: Deploy and manage microservices
- **CI/CD pipelines**: Automate build and deployment
- **Scalability**: Easy scaling of applications
- **Isolation**: Isolate application processes and resources

## Installation

```bash
# Ubuntu/Debian
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh
sudo usermod -aG docker $USER

# RHEL/CentOS
sudo yum install docker
sudo systemctl start docker
sudo usermod -aG docker $USER

# macOS
# Download Docker Desktop from https://www.docker.com/products/docker-desktop

# Windows
# Download Docker Desktop from https://www.docker.com/products/docker-desktop
```

## Basic Commands & Troubleshooting

### Essential Commands
```bash
# Check Docker version
docker --version

# Test Docker installation
docker run hello-world

# List running containers
docker ps

# List all containers (including stopped)
docker ps -a

# List images
docker images

# Pull image from registry
docker pull ubuntu:20.04

# Build image from Dockerfile
docker build -t image-name:tag .

# Run container
docker run image-name

# Run container with interactive terminal
docker run -it image-name /bin/bash

# Run container in background
docker run -d image-name

# Run container with port mapping
docker run -p 8080:80 image-name

# Run container with volume mounting
docker run -v /host/path:/container/path image-name

# Stop running container
docker stop container-id

# Start stopped container
docker start container-id

# Remove container
docker rm container-id

# Remove image
docker rmi image-name

# View container logs
docker logs container-id

# Execute command in running container
docker exec -it container-id /bin/bash

# Push image to registry
docker push username/image-name:tag

# Tag image
docker tag image-name username/image-name:tag
```

### Docker Compose
```bash
# Install Docker Compose
sudo curl -L "https://github.com/docker/compose/releases/download/v2.0.0/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose

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
docker-compose up -d

# Stop services
docker-compose down

# View logs
docker-compose logs

# Rebuild images
docker-compose build
```

### Common Issues & Resolution

**Issue: Cannot connect to Docker daemon**
```bash
# Solution: Start Docker service
sudo systemctl start docker

# Check if docker daemon is running
sudo systemctl status docker

# For non-root user access
sudo usermod -aG docker $USER
newgrp docker
```

**Issue: Permission denied**
```bash
# Solution: Add user to docker group
sudo usermod -aG docker $USER
# Log out and log back in
```

**Issue: Insufficient disk space**
```bash
# Solution: Clean up unused images and containers
docker system prune
docker system prune -a  # Also remove unused images

# Remove specific dangling images
docker image prune

# Check disk usage
docker system df
```

**Issue: Port already in use**
```bash
# Solution: Use different port mapping
docker run -p 8081:80 image-name

# Or kill process using port
sudo lsof -i :8080
sudo kill -9 <PID>
```

**Issue: Container crashes immediately**
```bash
# Solution: Check logs
docker logs container-id

# Run with interactive terminal to debug
docker run -it image-name /bin/bash
```

### Debugging
```bash
# Inspect container details
docker inspect container-id

# Check resource usage
docker stats

# View container processes
docker top container-id

# Attach to running container
docker attach container-id

# View image layers
docker history image-name

# Check Docker version and info
docker info
```

### Creating Custom Images
```bash
# Example Dockerfile
cat > Dockerfile << 'EOF'
FROM ubuntu:20.04
RUN apt-get update && apt-get install -y curl
COPY . /app
WORKDIR /app
CMD ["./app.sh"]
EOF

# Build image
docker build -t myapp:1.0 .

# View build details
docker build --verbose -t myapp:1.0 .
```

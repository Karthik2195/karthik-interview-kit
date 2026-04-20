# Docker - Senior Level Interview Questions & Answers

## Table of Contents
1. Image Optimization
2. Security & Best Practices
3. Performance & Scalability
4. Networking & Storage
5. Production Patterns

---

## 1. Image Optimization

### Q (High-Level): Why is my Docker image 2GB when my app is only 100MB?

**A (Real-Time Image Bloat):**
You build a Docker image for your Node.js app. It's huge!

**Common culprits:**
```dockerfile
# ❌ BAD: Image is 2GB
FROM ubuntu:latest  # 77MB
RUN apt-get update  # Downloads package lists: 200MB
RUN apt-get install -y build-essential  # 500MB
RUN apt-get install -y python3  # 300MB (why??)
COPY app.js .
RUN npm install  # Dependencies: 500MB
ENTRYPOINT ["node", "app.js"]

# Result: 2GB image for 100MB app! 😱
```

**Why this matters:**
- 2GB image takes 10 minutes to push/pull
- Slow deployments (everyone waits)
- Costs money (image registry storage)
- Slower pod startup (download large image first)

**Solution: Multi-stage build**
```dockerfile
# ✅ GOOD: Image is 50MB
# Stage 1: Build with tools
FROM node:18-alpine AS builder
COPY package*.json .
RUN npm install

# Stage 2: Runtime (lightweight)
FROM node:18-alpine
COPY --from=builder node_modules/ node_modules/
COPY app.js .
ENTRYPOINT ["node", "app.js"]

# Result: Only runtime deps, 50MB! ✅
```

**Real-time impact:**
- Before: Image push/pull = 10 minutes
- After: Image push/pull = 30 seconds
- Deployments 20x faster!

---

### Q (High-Level): Real-time scenario - Docker image has security vulnerability. How long until production is fixed?

**A (Security Incident):**
CVE alert: "OpenSSL in Ubuntu:latest has RCE vulnerability!"

**Without proper image strategy:**
```
Your current image (based on ubuntu:latest):
1. Old image already deployed in 500 containers
2. You need to rebuild everything
3. Retest every app
4. Redeploy 500 containers
5. Time: 3-4 hours while vulnerability active ⚠️
```

**With proper image strategy:**
```dockerfile
# ✅ Use specific versions, not latest
FROM ubuntu:22.04-LTS  # Specific version, not latest
RUN apt-get install openssl=1.1.1l-1ubuntu1.6  # Pinned version
```

**When vulnerability found:**
```
1. Update base image to ubuntu:22.04.2 (has patch)
2. Rebuild (1 minute)
3. Push to registry (10 seconds)
4. Kubernetes auto-redeploys pods (2 minutes)
5. All 500 containers patched
6. Time: 5 minutes total ✅
```

**Learning:**
- Never use `latest` tag in production
- Always pin specific versions
- Test image updates regularly

---

### Q: Design an optimized multi-stage Dockerfile for production

**A:**
```dockerfile
# Multi-stage build for optimal image size

# Stage 1: Build stage
FROM golang:1.20-alpine AS builder

WORKDIR /app

# Install dependencies
RUN apk add --no-cache git make

# Copy source
COPY . .

# Build application
RUN CGO_ENABLED=0 GOOS=linux go build \
    -ldflags="-w -s" \
    -o /app/myapp

---
# Stage 2: Runtime stage (minimal)
FROM alpine:3.18

# Install only runtime dependencies
RUN apk add --no-cache ca-certificates tzdata

# Create non-root user
RUN addgroup -D appgroup && adduser -D appuser -G appgroup

# Set labels
LABEL maintainer="karthik@example.com" \
      version="1.0.0" \
      description="Production application"

# Copy from builder
COPY --from=builder --chown=appuser:appgroup /app/myapp /usr/local/bin/myapp

# Health check
HEALTHCHECK --interval=30s --timeout=10s --start-period=5s --retries=3 \
    CMD /usr/local/bin/myapp -health

# Switch to non-root user
USER appuser

# Expose port
EXPOSE 8080

# Runtime arguments
ENTRYPOINT ["/usr/local/bin/myapp"]
CMD ["--port", "8080"]
```

**Image Size Comparison:**
```
Unoptimized: 850MB
With multi-stage: 15MB (98% reduction!)
```

**Optimization Techniques:**
1. Use smaller base images (alpine vs ubuntu)
2. Multi-stage builds
3. Minimize layers
4. Use .dockerignore
5. Non-root user
6. Health checks

---

### Q: How would you scan Docker images for vulnerabilities in CI/CD?

**A:**
```bash
#!/bin/bash
# Image scanning pipeline

# 1. Trivy scanner
scan_with_trivy() {
    local image="$1"
    
    trivy image \
        --severity HIGH,CRITICAL \
        --exit-code 1 \
        --no-progress \
        "$image"
}

# 2. Snyk scanner
scan_with_snyk() {
    local image="$1"
    
    snyk container test \
        --severity-threshold=high \
        "$image"
}

# 3. Anchore scanner
scan_with_anchore() {
    local image="$1"
    
    anchore-cli image add "$image"
    anchore-cli image wait "$image"
    anchore-cli image vuln "$image" all
}

# 4. Docker Scout
scan_with_docker_scout() {
    local image="$1"
    
    docker scout cves "$image" --format table
}

# Comprehensive scanning script
scan_image() {
    local image="$1"
    local report_file="scan_report.json"
    
    echo "Scanning image: $image"
    
    # Build image
    docker build -t "$image" .
    
    # Scan with multiple tools
    trivy image --format json -o "$report_file" "$image"
    
    # Parse results
    high_vulns=$(jq '[.Results[].Vulnerabilities[]? | select(.Severity=="HIGH")] | length' "$report_file")
    critical_vulns=$(jq '[.Results[].Vulnerabilities[]? | select(.Severity=="CRITICAL")] | length' "$report_file")
    
    echo "HIGH: $high_vulns, CRITICAL: $critical_vulns"
    
    if [[ $critical_vulns -gt 0 ]]; then
        echo "FAILED: Critical vulnerabilities found"
        exit 1
    fi
    
    if [[ $high_vulns -gt 5 ]]; then
        echo "FAILED: Too many high vulnerabilities"
        exit 1
    fi
}

# Usage in CI/CD
scan_image "myapp:latest"
```

---

## 2. Security & Best Practices

### Q: Design a secure container runtime architecture

**A:**
```yaml
# Docker Compose with security best practices

version: '3.9'

services:
  app:
    image: myapp:latest
    container_name: myapp
    
    # Security context
    security_opt:
      - no-new-privileges:true
    cap_drop:
      - ALL
    cap_add:
      - NET_BIND_SERVICE
    
    # Read-only root filesystem
    read_only: true
    tmpfs:
      - /tmp
      - /var/tmp
    
    # Resource limits
    mem_limit: 512m
    memswap_limit: 512m
    cpus: '1.0'
    cpu_shares: 1024
    
    # Healthcheck
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 10s
    
    # Restart policy
    restart_policy:
      condition: on-failure
      delay: 5s
      max_attempts: 3
      window: 120s
    
    # Network
    networks:
      - isolated
    
    # User
    user: "1001:1001"

  database:
    image: postgres:15-alpine
    security_opt:
      - no-new-privileges:true
    cap_drop:
      - ALL
    environment:
      POSTGRES_PASSWORD_FILE: /run/secrets/db_password
    secrets:
      - db_password
    volumes:
      - db_data:/var/lib/postgresql/data:Z
    networks:
      - isolated

volumes:
  db_data:
    driver: local

networks:
  isolated:
    driver: bridge

secrets:
  db_password:
    file: ./db_password.txt
```

---

### Q: How would you implement secrets management in Docker?

**A:**
```bash
# 1. Docker Secrets (Swarm Mode)
docker secret create db_password password.txt
docker secret inspect db_password

# 2. Environment Variables (NOT RECOMMENDED)
docker run -e DB_PASS=secret myapp  # BAD - visible in ps

# 3. Secret files (BETTER)
docker run -v /run/secrets/db_pass:/etc/secrets/db_pass:ro myapp

# 4. HashiCorp Vault integration
docker run \
  -e VAULT_ADDR=http://vault:8200 \
  -e VAULT_TOKEN=$VAULT_TOKEN \
  --mount type=bind,source=$(pwd)/vault-config,target=/vault/config \
  vault:latest

# 5. AWS Secrets Manager
docker run \
  -e AWS_REGION=us-east-1 \
  -e AWS_ACCESS_KEY_ID=$AWS_KEY \
  -e AWS_SECRET_ACCESS_KEY=$AWS_SECRET \
  myapp

# Recommended: Use Docker secrets in Compose
version: '3.9'
services:
  app:
    image: myapp
    secrets:
      - db_password
      - api_key
    environment:
      DB_PASSWORD_FILE: /run/secrets/db_password
      API_KEY_FILE: /run/secrets/api_key

secrets:
  db_password:
    file: ./secrets/db_password
  api_key:
    external: true  # From vault/external source
```

---

## 3. Performance & Scalability

### Q: Design a high-performance Docker registry setup

**A:**
```yaml
# Docker Registry with caching and optimization

version: '3.9'

services:
  registry:
    image: registry:2.8
    ports:
      - "5000:5000"
    volumes:
      - registry_data:/var/lib/registry
      - ./config.yml:/etc/docker/registry/config.yml:ro
    environment:
      REGISTRY_LOG_LEVEL: info
    networks:
      - registry-net

  # Redis cache for faster reads
  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data
    command: redis-server --maxmemory 2gb --maxmemory-policy allkeys-lru
    networks:
      - registry-net

  # Nginx as reverse proxy with caching
  nginx:
    image: nginx:alpine
    ports:
      - "443:443"
      - "80:80"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
      - ./certificates:/etc/nginx/certs:ro
      - nginx_cache:/var/cache/nginx
    depends_on:
      - registry
    networks:
      - registry-net

volumes:
  registry_data:
  redis_data:
  nginx_cache:

networks:
  registry-net:
```

**Registry Configuration:**
```yaml
# config.yml
version: 0.1
log:
  level: info
storage:
  cache:
    blobdescriptor: redis
  filesystem:
    rootdirectory: /var/lib/registry
auth:
  htpasswd:
    realm: Registry
    path: /etc/docker/registry/htpasswd
middleware:
  registry:
    - name: catalog
  repository:
    - name: url
  storage:
    - name: cloudfront
      options:
        baseurl: https://cdn.example.com
        privatekey: /etc/docker/registry/cloudfront.key
http:
  addr: :5000
  net: tcp
  keepalive: 120s
```

---

## 4. Networking & Storage

### Q: Design Docker networking for microservices

**A:**
```yaml
version: '3.9'

services:
  # API Gateway
  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
    networks:
      - external

  # Microservices
  user-service:
    image: myapp:user
    environment:
      DATABASE_URL: postgresql://db:5432/users
      CACHE_URL: redis://cache:6379
    depends_on:
      - db
      - cache
    networks:
      - internal
      - service-mesh

  order-service:
    image: myapp:order
    environment:
      USER_SERVICE_URL: http://user-service:8080
      DATABASE_URL: postgresql://db:5432/orders
    depends_on:
      - db
      - user-service
    networks:
      - internal
      - service-mesh

  # Infrastructure
  db:
    image: postgres:15-alpine
    environment:
      POSTGRES_MULTIPLE_DATABASES: users,orders,products
    volumes:
      - db_data:/var/lib/postgresql/data
    networks:
      - internal

  cache:
    image: redis:7-alpine
    networks:
      - internal

  # Service mesh (optional)
  jaeger:
    image: jaegertracing/all-in-one
    ports:
      - "16686:16686"
    networks:
      - service-mesh

networks:
  external:
    driver: bridge
  internal:
    driver: bridge
    driver_opts:
      com.docker.network.bridge.enable_ip_masquerade: 'true'
  service-mesh:
    driver: overlay

volumes:
  db_data:
```

---

### Q: How would you manage persistent storage in Docker?

**A:**
```bash
# 1. Named volumes (recommended)
docker volume create mydata
docker run -v mydata:/data myapp

# 2. Bind mounts
docker run -v /host/path:/container/path myapp

# 3. Tmpfs mounts
docker run --tmpfs /tmp:rw,size=1024m myapp

# 4. NFS volumes
docker volume create \
  --driver local \
  --opt type=nfs \
  --opt o=addr=nfs-server,vers=4,soft,timeo=180,bg,tcp \
  --opt device=:/export/path \
  nfs-volume

# 5. Docker Compose with volumes
version: '3.9'
services:
  app:
    image: myapp
    volumes:
      - data:/app/data
      - logs:/var/log
      - ./config:/etc/config:ro

  db:
    image: postgres
    volumes:
      - db_data:/var/lib/postgresql/data
    environment:
      PGDATA: /var/lib/postgresql/data/pgdata

volumes:
  data:
    driver: local
  logs:
    driver: local
  db_data:
    driver: local
    driver_opts:
      type: tmpfs
      device: tmpfs
      o: size=100m,uid=1000
```

---

## 5. Production Patterns

### Q: Design a blue-green deployment with Docker Compose

**A:**
```bash
#!/bin/bash
# Blue-green deployment script

BLUE_PORT=8000
GREEN_PORT=8001
CURRENT_PORT=""

check_health() {
    local port=$1
    local retries=30
    
    for i in $(seq 1 $retries); do
        if curl -sf http://localhost:$port/health > /dev/null; then
            return 0
        fi
        sleep 1
    done
    return 1
}

switch_traffic() {
    local port=$1
    
    docker-compose exec nginx \
        sed -i "s/server .*:.*:/server localhost:$port:/" /etc/nginx/nginx.conf
    
    docker-compose exec nginx nginx -s reload
}

deploy_blue_green() {
    local version="$1"
    
    # Determine which is current
    CURRENT_PORT=$BLUE_PORT
    NEW_PORT=$GREEN_PORT
    
    echo "Deploying $version to port $NEW_PORT"
    
    # Deploy new version
    docker-compose up -d --no-deps app-green
    docker-compose exec app-green docker pull myapp:$version
    
    # Wait for health checks
    if ! check_health $NEW_PORT; then
        echo "Health check failed"
        docker-compose down app-green
        return 1
    fi
    
    # Run tests
    if ! run_smoke_tests $NEW_PORT; then
        echo "Tests failed"
        docker-compose down app-green
        return 1
    fi
    
    # Switch traffic
    switch_traffic $NEW_PORT
    
    # Clean up old version
    docker-compose down app-blue
    
    echo "Deployment successful"
}

run_smoke_tests() {
    local port=$1
    
    curl -f http://localhost:$port/api/health || return 1
    curl -f http://localhost:$port/api/test || return 1
}

deploy_blue_green "1.1.0"
```

---

## Summary

**Critical Senior Docker Skills:**
1. ✅ Multi-stage builds
2. ✅ Image optimization & security scanning
3. ✅ Security best practices
4. ✅ Secrets management
5. ✅ High-performance registry
6. ✅ Networking architectures
7. ✅ Storage management
8. ✅ Production deployment patterns

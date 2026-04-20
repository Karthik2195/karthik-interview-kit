# GitLab - Git Repository & CI/CD Platform

## What is it used for?
GitLab is used for:
- **Git repository hosting**: Self-hosted or cloud-based
- **CI/CD pipelines**: Integrated continuous integration
- **Code review**: Merge requests for quality control
- **Project management**: Issues, milestones, planning boards
- **Wiki documentation**: Built-in documentation
- **Security scanning**: SAST, DAST, dependency scanning
- **Container registry**: Built-in Docker registry
- **DevOps features**: Release management, deployment tracking

## Installation

```bash
# Ubuntu/Debian - Install GitLab
curl https://packages.gitlab.com/install/repositories/gitlab/gitlab-ee/script.deb.sh | sudo bash
sudo apt-get install gitlab-ee

# RHEL/CentOS
curl https://packages.gitlab.com/install/repositories/gitlab/gitlab-ee/script.rpm.sh | sudo bash
sudo yum install gitlab-ee

# Docker
docker run --detach \
  --hostname gitlab.example.com \
  --publish 443:443 --publish 80:80 --publish 22:22 \
  --name gitlab \
  --restart always \
  --volume /srv/gitlab/config:/etc/gitlab \
  --volume /srv/gitlab/logs:/var/log/gitlab \
  --volume /srv/gitlab/data:/var/opt/gitlab \
  gitlab/gitlab-ee:latest

# macOS
brew install gitlab-runner

# Verify installation
gitlab-ctl status

# Access GitLab
# Browser: http://localhost
# Default: root / (will prompt for password on first login)
```

## Basic Commands & Troubleshooting

### Essential Commands
```bash
# Check GitLab status
sudo gitlab-ctl status

# Start GitLab
sudo gitlab-ctl start

# Stop GitLab
sudo gitlab-ctl stop

# Restart GitLab
sudo gitlab-ctl restart

# Reconfigure GitLab
sudo gitlab-ctl reconfigure

# Check logs
sudo gitlab-ctl tail

# View specific component logs
sudo gitlab-ctl tail nginx
sudo gitlab-ctl tail postgresql

# Create user via CLI
sudo gitlab-rails console production
# Then in console:
# User.create(username: 'newuser', email: 'user@example.com', password: 'password123')

# Reset admin password
sudo gitlab-rails console production
# User.find_by(username: 'root').reset_password

# Create personal access token
# UI: Settings -> Access Tokens -> New token

# Clone repository
git clone https://gitlab.example.com/username/repo.git
git clone git@gitlab.example.com:username/repo.git

# Create new project
# UI: New project or use API
```

### GitLab CI/CD - .gitlab-ci.yml
```yaml
# .gitlab-ci.yml
variables:
  DOCKER_IMAGE: nginx:latest

stages:
  - build
  - test
  - deploy

before_script:
  - echo "Starting job"

build_job:
  stage: build
  script:
    - docker build -t myapp:latest .
    - docker push registry.gitlab.com/username/myapp:latest
  only:
    - main

test_job:
  stage: test
  image: node:18
  script:
    - npm install
    - npm test
  coverage: '/Coverage: \d+\.\d+%/'
  artifacts:
    reports:
      coverage_report:
        coverage_format: cobertura
        path: coverage/cobertura-coverage.xml

deploy_job:
  stage: deploy
  image: kubectl:latest
  script:
    - kubectl set image deployment/myapp myapp=registry.gitlab.com/username/myapp:latest
  only:
    - main
  environment:
    name: production
```

### GitLab Runner Setup
```bash
# Install GitLab Runner
# macOS
brew install gitlab-runner

# Linux
curl -L https://packages.gitlab.com/install/repositories/runner/gitlab-runner/script.deb.sh | sudo bash
sudo apt-get install gitlab-runner

# Windows
# Download from https://gitlab-runner-downloads.s3.amazonaws.com/latest/binaries/gitlab-runner-windows-amd64.exe

# Register runner
sudo gitlab-runner register \
  --url https://gitlab.example.com \
  --registration-token RUNNER_REGISTRATION_TOKEN \
  --executor shell \
  --description "My Runner"

# Or interactive registration
sudo gitlab-runner register

# Start runner
sudo gitlab-runner start

# List registered runners
gitlab-runner list

# Verify runner can connect
sudo gitlab-runner verify
```

### GitLab API
```bash
# Get user info
curl --header "PRIVATE-TOKEN: YOUR_TOKEN" \
  https://gitlab.example.com/api/v4/user

# List projects
curl --header "PRIVATE-TOKEN: YOUR_TOKEN" \
  https://gitlab.example.com/api/v4/projects

# Create project
curl --request POST \
  --header "PRIVATE-TOKEN: YOUR_TOKEN" \
  -d "name=new-project&visibility=private" \
  https://gitlab.example.com/api/v4/projects

# Get project details
curl --header "PRIVATE-TOKEN: YOUR_TOKEN" \
  https://gitlab.example.com/api/v4/projects/:id

# List issues
curl --header "PRIVATE-TOKEN: YOUR_TOKEN" \
  https://gitlab.example.com/api/v4/projects/:id/issues

# Create issue
curl --request POST \
  --header "PRIVATE-TOKEN: YOUR_TOKEN" \
  -d "title=New Issue&description=Description" \
  https://gitlab.example.com/api/v4/projects/:id/issues

# List merge requests
curl --header "PRIVATE-TOKEN: YOUR_TOKEN" \
  https://gitlab.example.com/api/v4/projects/:id/merge_requests

# Get pipeline status
curl --header "PRIVATE-TOKEN: YOUR_TOKEN" \
  https://gitlab.example.com/api/v4/projects/:id/pipelines
```

### Common Issues & Resolution

**Issue: Runner doesn't pick up jobs**
```bash
# Solution: Check runner status
sudo gitlab-runner verify

# Check runner configuration
cat /etc/gitlab-runner/config.toml

# Verify executor
sudo gitlab-runner list

# Check connectivity
curl -v https://gitlab.example.com/api/v4/runners/verify \
  -H "PRIVATE-TOKEN: YOUR_TOKEN"

# Restart runner
sudo gitlab-runner restart
```

**Issue: Pipeline stuck in pending**
```bash
# Solution: Ensure runners are active
# UI: Admin -> Runners

# Check runner availability
sudo gitlab-runner status

# Verify runner has correct tags
cat /etc/gitlab-runner/config.toml

# Restart runner service
sudo systemctl restart gitlab-runner

# Check executor capability
sudo gitlab-runner list
```

**Issue: CI/CD environment variables not accessible**
```bash
# Solution: Define variables in .gitlab-ci.yml
variables:
  MY_VAR: "value"

# Or in project settings
# UI: Project -> Settings -> CI/CD -> Variables

# Access in script
script:
  - echo $MY_VAR

# Check masked variables (don't print them)
# UI: Mark as "Masked" in variable settings
```

**Issue: Docker registry authentication fails**
```bash
# Solution: Create deploy token
# UI: Project -> Settings -> Repository -> Deploy tokens

# Login to registry
docker login registry.gitlab.com -u username -p token

# Or use CI/CD credentials
# In runner environment

# Verify credentials
docker login -u username registry.gitlab.com
```

**Issue: SSH authentication fails**
```bash
# Solution: Generate SSH key
ssh-keygen -t ed25519 -C "user@example.com"

# Add public key to GitLab
# UI: Settings -> SSH Keys

# Test connection
ssh -T git@gitlab.example.com

# Configure git with SSH
git clone git@gitlab.example.com:username/repo.git
```

### Debugging

```bash
# Check GitLab service status
sudo gitlab-ctl status

# View system logs
sudo gitlab-ctl tail

# Check specific service logs
sudo gitlab-ctl tail gitlab-rails
sudo gitlab-ctl tail nginx

# Database connection
sudo gitlab-rails db:migrate:status

# Check disk usage
sudo gitlab-ctl reconfigure --only package

# Verify all components
sudo gitlab-ctl reconfigure

# Check runner connection
sudo gitlab-runner verify

# Show runner debug info
sudo gitlab-runner -debug register
```

### Container Registry

```bash
# Build and push image
docker build -t registry.gitlab.com/username/project:latest .
docker login registry.gitlab.com
docker push registry.gitlab.com/username/project:latest

# Pull image
docker pull registry.gitlab.com/username/project:latest

# Deploy from registry
# In .gitlab-ci.yml
image: registry.gitlab.com/username/project:latest
```

### Project Management Features

**Create Issue**
```bash
# Via UI: Project -> Issues -> New issue

# Via API
curl --request POST \
  --header "PRIVATE-TOKEN: TOKEN" \
  -d "title=Bug&description=Description&labels=bug" \
  https://gitlab.example.com/api/v4/projects/:id/issues
```

**Create Merge Request**
```bash
# Create branch
git checkout -b feature/new-feature

# Make changes and commit
git commit -am "Add new feature"

# Push branch
git push origin feature/new-feature

# Create MR via UI or API
curl --request POST \
  --header "PRIVATE-TOKEN: TOKEN" \
  -d "source_branch=feature/new-feature&target_branch=main&title=New Feature" \
  https://gitlab.example.com/api/v4/projects/:id/merge_requests
```

### Security Features

**Enable SAST (Static Application Security Testing)**
```yaml
# .gitlab-ci.yml
stages:
  - security
  - test

sast:
  stage: security
  include:
    - template: Security/SAST.gitlab-ci.yml
```

**Enable DAST (Dynamic Application Security Testing)**
```yaml
dast:
  stage: security
  include:
    - template: Security/DAST.gitlab-ci.yml
  variables:
    DAST_WEBSITE: https://application.example.com
```

**Enable Dependency Scanning**
```yaml
dependency_scanning:
  stage: security
  include:
    - template: Security/Dependency-Scanning.gitlab-ci.yml
```

### Advanced Configurations

**Schedule pipelines**
```bash
# UI: Project -> Schedules -> New schedule
# Set description, interval, and branch
```

**Deploy to production with approval**
```yaml
deploy_production:
  stage: deploy
  script:
    - kubectl apply -f deployment.yaml
  environment:
    name: production
  when: manual
  only:
    - main
```

**Matrix builds (run job with multiple configurations)**
```yaml
build_matrix:
  stage: build
  script:
    - echo "Building $RUBY_VERSION"
  parallel:
    matrix:
      - RUBY_VERSION: ["2.7", "3.0", "3.1"]
```

### Performance Monitoring

```bash
# Check GitLab performance
# UI: Admin -> System Health -> Monitoring

# View database performance
sudo gitlab-rails console production
# ActiveRecord::Base.logger = Logger.new(STDOUT)

# Monitor runner performance
sudo gitlab-runner list

# Check server resources
free -h  # Memory
df -h    # Disk space
```

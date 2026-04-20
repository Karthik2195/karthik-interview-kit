# Bitbucket - Git Repository & CI/CD Platform

## What is it used for?
Bitbucket is used for:
- **Git hosting**: Private and public repositories
- **Code review**: Pull requests for quality
- **CI/CD pipelines**: Integrated Bitbucket Pipelines
- **Team collaboration**: Team-based development
- **Branch permissions**: Enforce branching strategies
- **Integrations**: Connect to 3rd party tools
- **Webhooks**: Automated workflows
- **Jira integration**: Connect to issue tracking

## Installation & Setup

```bash
# Install Git (required)
sudo apt-get install git

# Generate SSH key
ssh-keygen -t ed25519 -C "user@example.com"

# Add SSH key to Bitbucket
# UI: Personal Settings -> SSH Keys

# Clone repository
git clone git@bitbucket.org:workspace/repo.git
git clone https://bitbucket.org/workspace/repo.git

# Configure Bitbucket CLI
pip install atlassian-python-api
```

## Basic Commands & Troubleshooting

### Essential Commands
```bash
# Clone repository
git clone git@bitbucket.org:workspace/repo.git

# Create branch
git checkout -b feature/new-feature

# Commit changes
git add .
git commit -m "Add new feature"

# Push to Bitbucket
git push origin feature/new-feature

# Create pull request
# Web UI -> Create Pull Request

# Merge pull request
# Web UI -> Pull Request -> Merge

# Delete branch
git push origin --delete feature/new-feature

# View pull requests
# Web UI -> Pull Requests

# View commits
# Web UI -> Commits

# View branches
git branch -a
```

### Bitbucket Pipelines (CI/CD)
```yaml
# bitbucket-pipelines.yml
image: node:18

pipelines:
  default:
    - step:
        name: Build & Test
        script:
          - npm install
          - npm test
          - npm run build

  branches:
    main:
      - step:
          name: Build
          script:
            - npm install
            - npm run build
      - step:
          name: Deploy
          script:
            - docker build -t myapp:latest .
            - docker push registry/myapp:latest

    develop:
      - step:
          name: Build & Test
          script:
            - npm install
            - npm test

definitions:
  caches:
    npm: ~/.npm
  services:
    docker:
      memory: 3076

options:
  docker: true
```

### Common Issues & Resolution

**Issue: SSH authentication fails**
```bash
# Solution: Verify SSH key setup
ssh -T git@bitbucket.org

# Generate new SSH key
ssh-keygen -t ed25519 -C "user@bitbucket.org"

# Add to SSH agent
ssh-add ~/.ssh/id_ed25519

# Update SSH config
cat >> ~/.ssh/config << 'EOF'
Host bitbucket.org
    HostName bitbucket.org
    User git
    IdentityFile ~/.ssh/id_ed25519
EOF
```

**Issue: Pipeline not running**
```bash
# Solution: Enable Pipelines
# UI: Repository -> Settings -> Pipelines

# Check pipeline configuration
cat bitbucket-pipelines.yml

# Validate YAML
yamllint bitbucket-pipelines.yml

# View pipeline runs
# UI: Repository -> Pipelines -> Runs
```

**Issue: Merge conflict**
```bash
# Solution: Resolve locally
git fetch origin
git merge origin/main
# Manually resolve conflicts
git add .
git commit -m "Resolve merge conflict"
git push origin feature/branch
```

### Debugging
```bash
# Enable debug mode
git -c core.askpass=true clone ...

# Check remote URL
git remote -v

# View git log
git log --oneline

# Check git status
git status

# View recent commits
git log -n 5 --oneline

# View diff
git diff HEAD~1

# Check SSH connection
ssh -Tv git@bitbucket.org
```

### Webhooks
```bash
# Create webhook
# UI: Repository -> Settings -> Webhooks -> Add

# Common events:
# - Push commits
# - Pull request created
# - Pull request updated
# - Issue created
```

### Branch Permissions
```bash
# Set branch restrictions
# UI: Repository -> Settings -> Branch Permissions

# Require pull request reviews
# UI: Settings -> Pull requests -> Required reviewers

# Require passing builds
# UI: Settings -> Pull requests -> Require passing builds
```

### Integrations
```bash
# Jira integration
# UI: Repository -> Settings -> Connected Repositories

# Slack integration
# UI: Settings -> Integrations -> Slack

# Deploy to production
# UI: Deployments -> Create deployment
```

### Repository Settings
```bash
# Rename repository
# UI: Repository -> Settings -> Repository Name

# Transfer repository
# UI: Repository -> Settings -> Transfer repository

# Archive repository
# UI: Repository -> Settings -> Make Private

# Delete repository
# UI: Repository -> Settings -> Delete
```

### Advanced Features
```bash
# Use default reviewers
# UI: Settings -> Default reviewers

# Require resolution of comments
# UI: Settings -> Pull requests

# Enforce merge strategies
# UI: Settings -> Merge strategies

# Use branch models
# UI: Settings -> Branching model
```

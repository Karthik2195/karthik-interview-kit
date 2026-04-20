# GitHub - Version Control & Collaboration Platform

## What is it used for?
GitHub is used for:
- **Version control**: Host Git repositories
- **Collaboration**: Team-based code development
- **Code review**: Pull requests for code quality
- **Issue tracking**: Bug reports and feature requests
- **CI/CD integration**: GitHub Actions for automation
- **Documentation**: README files and wikis
- **Open source**: Community-driven development
- **Security**: Vulnerability scanning and secrets management

## Installation & Setup

```bash
# Install Git
# Ubuntu/Debian
sudo apt-get install git

# macOS
brew install git

# Windows
# Download from https://git-scm.com/

# Verify installation
git --version

# Configure Git
git config --global user.name "Your Name"
git config --global user.email "your.email@example.com"

# Generate SSH key
ssh-keygen -t ed25519 -C "your.email@example.com"

# Add SSH key to GitHub
# 1. Copy public key: cat ~/.ssh/id_ed25519.pub
# 2. Go to: GitHub Settings -> SSH and GPG keys -> New SSH key
# 3. Paste key and save
```

## Basic Commands & Troubleshooting

### Essential Commands
```bash
# Clone repository
git clone https://github.com/username/repo.git
git clone git@github.com:username/repo.git  # SSH

# Navigate to repo
cd repo

# Check status
git status

# View differences
git diff

# Add files to staging
git add .
git add filename.txt

# Commit changes
git commit -m "Commit message"

# Push to remote
git push origin branch-name

# Push to main branch
git push origin main

# Pull from remote
git pull origin branch-name

# Fetch without merging
git fetch origin

# Create new branch
git branch new-branch

# Switch branch
git checkout new-branch
git switch new-branch

# Create and switch branch
git checkout -b new-branch
git switch -c new-branch

# List branches
git branch -a

# Delete local branch
git branch -d branch-name

# Delete remote branch
git push origin --delete branch-name

# Merge branch to current
git merge branch-name

# Rebase current on main
git rebase main

# View commit history
git log

# View specific commit
git show commit-hash

# Undo last commit (keep changes)
git reset --soft HEAD~1

# Undo last commit (discard changes)
git reset --hard HEAD~1

# Stash changes
git stash

# Apply stashed changes
git stash pop

# View remote URLs
git remote -v

# Add remote
git remote add origin https://github.com/username/repo.git
```

### GitHub REST API
```bash
# Get user info
curl https://api.github.com/users/username

# List repositories
curl https://api.github.com/users/username/repos

# Create repository (requires authentication)
curl -X POST -H "Authorization: token YOUR_TOKEN" \
  -d '{"name":"new-repo","description":"Description"}' \
  https://api.github.com/user/repos

# List issues
curl https://api.github.com/repos/username/repo/issues

# Create issue
curl -X POST -H "Authorization: token YOUR_TOKEN" \
  -d '{"title":"Issue Title","body":"Issue description"}' \
  https://api.github.com/repos/username/repo/issues

# Get pull requests
curl https://api.github.com/repos/username/repo/pulls

# Create pull request
curl -X POST -H "Authorization: token YOUR_TOKEN" \
  -d '{
    "title":"PR Title",
    "head":"feature-branch",
    "base":"main"
  }' \
  https://api.github.com/repos/username/repo/pulls

# Get repository details
curl https://api.github.com/repos/username/repo

# List releases
curl https://api.github.com/repos/username/repo/releases

# Create release
curl -X POST -H "Authorization: token YOUR_TOKEN" \
  -d '{
    "tag_name":"v1.0.0",
    "name":"Version 1.0.0",
    "body":"Release notes"
  }' \
  https://api.github.com/repos/username/repo/releases
```

### GitHub Actions (CI/CD)
```yaml
# .github/workflows/test.yml
name: Tests

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]

jobs:
  test:
    runs-on: ubuntu-latest

    steps:
    - uses: actions/checkout@v3
    
    - name: Set up Node.js
      uses: actions/setup-node@v3
      with:
        node-version: '18'
    
    - name: Install dependencies
      run: npm install
    
    - name: Run tests
      run: npm test
    
    - name: Run linter
      run: npm run lint
    
    - name: Build
      run: npm run build
```

### Common Issues & Resolution

**Issue: SSH key permission denied**
```bash
# Solution: Set correct permissions
chmod 700 ~/.ssh
chmod 600 ~/.ssh/id_ed25519
chmod 644 ~/.ssh/id_ed25519.pub

# Test SSH connection
ssh -T git@github.com

# Add key to agent
ssh-add ~/.ssh/id_ed25519
```

**Issue: Authentication failed (HTTPS)**
```bash
# Solution: Use personal access token
# Create token: GitHub Settings -> Developer settings -> Personal access tokens

# Store credentials
git config --global credential.helper store
git push origin main
# Enter username and token when prompted

# Or use credential manager
git config --global credential.helper manager-core
```

**Issue: Merge conflict**
```bash
# Solution: Resolve conflicts manually
git status  # See conflicted files

# Edit conflicted files (look for <<<, ===, >>>)
# After resolving:
git add .
git commit -m "Resolve merge conflict"
git push origin branch-name

# Or abort merge
git merge --abort
```

**Issue: Accidentally committed sensitive data**
```bash
# Solution: Use git-filter-repo (if not yet pushed)
git reset HEAD~1
# Remove file
git rm --cached sensitive-file.txt
git add .gitignore  # Add to .gitignore
git commit -m "Remove sensitive data"

# If already pushed to public repo
# 1. Create new token to invalidate old one
# 2. Use BFG Repo-Cleaner to remove from history
# 3. Force push: git push --force-with-lease
```

**Issue: Large files causing issues**
```bash
# Solution: Use Git LFS (Large File Storage)
brew install git-lfs  # macOS
apt-get install git-lfs  # Linux

# Track file type
git lfs track "*.psd"

# Add and commit
git add .gitattributes
git commit -m "Add LFS tracking"

# Push large files
git push origin main
```

### Debugging
```bash
# Check SSH connection
ssh -T git@github.com -v

# Check Git configuration
git config --list

# Check global configuration
cat ~/.gitconfig

# Test repository connectivity
git ls-remote origin

# Check available refs
git for-each-ref

# See what will be pushed
git push --dry-run origin branch-name

# View reflog (history of HEAD)
git reflog

# Check if commit exists
git branch --contains commit-hash

# Find which branch contains commit
git log --all --oneline | grep "commit message"
```

### GitHub CLI (gh)
```bash
# Install GitHub CLI
brew install gh  # macOS
apt-get install gh  # Linux

# Authenticate
gh auth login

# Clone repository
gh repo clone username/repo

# Create repository
gh repo create my-repo

# List repositories
gh repo list

# Create issue
gh issue create --title "Issue title" --body "Issue description"

# List issues
gh issue list

# Create pull request
gh pr create --title "PR title" --body "PR description"

# List pull requests
gh pr list

# Check pull request status
gh pr status

# View PR
gh pr view <number>

# Merge PR
gh pr merge <number>
```

### Common Workflows
```bash
# Feature branch workflow
git checkout -b feature/new-feature
# Make changes
git add .
git commit -m "Add new feature"
git push origin feature/new-feature
# Create PR on GitHub

# Sync fork with upstream
git remote add upstream https://github.com/original/repo.git
git fetch upstream
git checkout main
git merge upstream/main
git push origin main

# Update PR with latest changes
git fetch origin
git rebase origin/main
git push origin feature-branch --force-with-lease
```

# Jenkins - CI/CD Automation Server

## What is it used for?
Jenkins is used for:
- **Continuous Integration**: Automatically test code changes
- **Continuous Deployment**: Automate software deployment
- **Build automation**: Automate build processes
- **Testing**: Run automated tests on code changes
- **Artifact management**: Store and manage build artifacts
- **Pipeline orchestration**: Define complex deployment workflows
- **Monitoring**: Monitor build and deployment status

## Installation

```bash
# Ubuntu/Debian
wget -q -O - https://pkg.jenkins.io/debian-stable/jenkins.io.key | sudo apt-key add -
sudo sh -c 'echo deb https://pkg.jenkins.io/debian-stable binary/ > /etc/apt/sources.list.d/jenkins.list'
sudo apt-get update
sudo apt-get install jenkins

# RHEL/CentOS
sudo wget -O /etc/yum.repos.d/jenkins.repo https://pkg.jenkins.io/redhat-stable/jenkins.repo
sudo rpm --import https://pkg.jenkins.io/redhat-stable/jenkins.io.key
sudo yum upgrade -y
sudo yum install java-11-openjdk java-11-openjdk-devel
sudo yum install jenkins

# Start Jenkins
sudo systemctl start jenkins
sudo systemctl enable jenkins

# Check if running
curl http://localhost:8080

# Get initial admin password
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```

## Basic Commands & Troubleshooting

### Essential Commands
```bash
# Start Jenkins
sudo systemctl start jenkins

# Stop Jenkins
sudo systemctl stop jenkins

# Restart Jenkins
sudo systemctl restart jenkins

# Check status
sudo systemctl status jenkins

# View logs
sudo tail -f /var/log/jenkins/jenkins.log

# Access Jenkins UI
# Browser: http://localhost:8080

# Jenkins CLI
java -jar jenkins-cli.jar -s http://localhost:8080 help

# Create job
java -jar jenkins-cli.jar -s http://localhost:8080 create-job my-job < job-config.xml

# List jobs
java -jar jenkins-cli.jar -s http://localhost:8080 list-jobs

# Build job
java -jar jenkins-cli.jar -s http://localhost:8080 build my-job

# Get build log
java -jar jenkins-cli.jar -s http://localhost:8080 console my-job 1

# Get job info
java -jar jenkins-cli.jar -s http://localhost:8080 get-job my-job
```

### Jenkins Pipeline Example (Jenkinsfile)
```groovy
pipeline {
    agent any
    
    environment {
        NODE_ENV = 'production'
        BUILD_VERSION = "1.0.${BUILD_NUMBER}"
    }
    
    stages {
        stage('Checkout') {
            steps {
                git 'https://github.com/example/repo.git'
            }
        }
        
        stage('Install') {
            steps {
                sh 'npm install'
            }
        }
        
        stage('Test') {
            steps {
                sh 'npm test'
            }
        }
        
        stage('Build') {
            steps {
                sh 'npm run build'
            }
        }
        
        stage('Deploy') {
            when {
                branch 'main'
            }
            steps {
                sh 'docker build -t myapp:${BUILD_VERSION} .'
                sh 'docker push myapp:${BUILD_VERSION}'
                sh 'kubectl set image deployment/myapp myapp=myapp:${BUILD_VERSION}'
            }
        }
    }
    
    post {
        always {
            cleanWs()
        }
        success {
            echo 'Pipeline succeeded!'
        }
        failure {
            echo 'Pipeline failed!'
            // Send notification
        }
    }
}
```

### Freestyle Job Configuration
```xml
<?xml version="1.0" encoding="UTF-8"?>
<project>
  <description>Example freestyle job</description>
  <scm class="hudson.plugins.git.GitSCM">
    <userRemoteConfigs>
      <hudson.plugins.git.UserRemoteConfig>
        <url>https://github.com/example/repo.git</url>
      </hudson.plugins.git.UserRemoteConfig>
    </userRemoteConfigs>
    <branches>
      <hudson.plugins.git.BranchSpec>
        <name>*/main</name>
      </hudson.plugins.git.BranchSpec>
    </branches>
  </scm>
  <triggers>
    <com.cloudbees.jenkins.GitHubPushTrigger>
      <spec></spec>
    </com.cloudbees.jenkins.GitHubPushTrigger>
  </triggers>
  <builders>
    <hudson.tasks.Shell>
      <command>npm install && npm test</command>
    </hudson.tasks.Shell>
  </builders>
</project>
```

### Common Issues & Resolution

**Issue: Jenkins won't start**
```bash
# Solution: Check Java installation
java -version

# Check port 8080 availability
sudo lsof -i :8080

# Check logs
sudo tail -f /var/log/jenkins/jenkins.log

# Try starting with explicit Java
sudo /usr/bin/java -jar /usr/share/jenkins/jenkins.war
```

**Issue: Build fails with permission denied**
```bash
# Solution: Run Jenkins as appropriate user
sudo chown jenkins:jenkins /path/to/workspace

# Add user to docker group
sudo usermod -aG docker jenkins
sudo systemctl restart jenkins

# For privileged operations, use sudo
# Configure sudoers: sudo visudo
# jenkins ALL=(ALL) NOPASSWD: ALL
```

**Issue: Git authentication failed**
```bash
# Solution: Generate SSH key for jenkins user
sudo su - jenkins
ssh-keygen -t rsa -N "" -f ~/.ssh/id_rsa

# Add public key to GitHub
cat ~/.ssh/id_rsa.pub

# Or use credentials in Jenkins
# Manage Jenkins -> Manage Credentials -> Add

# Test connection
ssh -T git@github.com
```

**Issue: Plugin compatibility issues**
```bash
# Solution: Update Jenkins and plugins
# Manage Jenkins -> Plugin Manager -> Updates

# Or use CLI
java -jar jenkins-cli.jar -s http://localhost:8080 install-plugin PLUGIN_NAME

# List installed plugins
java -jar jenkins-cli.jar -s http://localhost:8080 list-plugins

# Restart after updates
sudo systemctl restart jenkins
```

**Issue: Disk space full**
```bash
# Solution: Clean workspace
java -jar jenkins-cli.jar -s http://localhost:8080 quiet-down
java -jar jenkins-cli.jar -s http://localhost:8080 cancel-quiet-down

# Remove old builds
# Job Configuration -> Discard old builds

# Check disk usage
du -sh /var/lib/jenkins/workspace/*

# Archive and compress old builds
cd /var/lib/jenkins/jobs/JOBNAME/builds/
tar -czf builds-backup.tar.gz old-build-numbers/
```

### Debugging
```bash
# Enable debug logging
# In Jenkins UI: Manage Jenkins -> Configure System -> Debug
export JENKINS_DEBUG_LEVEL=java.util.logging.Level.FINE

# Check Jenkins startup logs
sudo journalctl -u jenkins -n 50

# Test Jenkins CLI
java -jar jenkins-cli.jar -s http://localhost:8080 version

# Test credentials
java -jar jenkins-cli.jar -s http://localhost:8080 who-am-i

# Get system info
java -jar jenkins-cli.jar -s http://localhost:8080 get-node-config master

# Check job configuration
java -jar jenkins-cli.jar -s http://localhost:8080 get-job job-name > job-config.xml
```

### Jenkins Configuration as Code (JCasC)
```yaml
# jenkins.yaml
jenkins:
  systemMessage: "Jenkins CI Server"
  numExecutors: 4
  scm:
    git:
      globalConfigName: "Jenkins"
      globalConfigEmail: "jenkins@example.com"
  
credentials:
  system:
    domainCredentials:
      - credentials:
          - basic:
              scope: GLOBAL
              id: "github-credentials"
              username: "user"
              password: "password"

unclassified:
  location:
    url: "http://jenkins.example.com/"

tool:
  git:
    installations:
      - name: "git"
        home: "/usr/bin/git"
  maven:
    installations:
      - name: "maven"
        home: "/usr/share/maven"
```

### Common Plugins
```bash
# Install essential plugins
java -jar jenkins-cli.jar -s http://localhost:8080 install-plugin \
  git github docker kubernetes pipeline

# List installed
java -jar jenkins-cli.jar -s http://localhost:8080 list-plugins
```

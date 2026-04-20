# Jenkins - Senior Level Interview Questions & Answers

## Table of Contents
1. Pipeline Architecture
2. Performance & Scalability
3. Security & RBAC
4. Integration & Automation
5. Disaster Recovery

---

## 1. Pipeline Architecture

### Q (High-Level): What does Jenkins do? Why not just run scripts manually?

**A (Real-Time CI/CD):**
Scenario: Every developer push requires: test → build → deploy

**Manual approach:**
```
Developer pushes code:
1. Developer SSH into build server
2. Run tests manually
3. Wait 10 minutes
4. If pass, run build
5. If build works, deploy
6. Tell operations to restart servers
= 30 minutes per release
```

**With Jenkins:**
```
Developer pushes code:
1. Jenkins detects push (webhook)
2. Jenkins auto-runs pipeline:
   - Run tests (10 min)
   - Build artifact (5 min)
   - Deploy to staging (5 min)
   - Run smoke tests (2 min)
   - Wait for approval (human click button)
   - Deploy to production (5 min)
3. Total: 27 min automated, 1 min human = 28 min end-to-end
```

**Real-time benefit:**
- Manual: 10 deploys/day max, error-prone
- Jenkins: 100+ deploys/day, consistent, zero-human-error

---

### Q (High-Level): Real-time scenario - Code pushed at 5 PM Friday, deploy fails, it's critical bug. How do you revert?

**A (Weekend Fire at 5 PM):**
```
5:00 PM Friday: Deploy to production
5:03 PM: Customers report "checkout broken!"
5:05 PM: Realize deployment introduced bug
5:10 PM: Need to revert ASAP
```

**Without Jenkins:**
```
Manual revert:
- SSH to server
- Find previous version
- Copy files
- Restart services
- Cross fingers
Time: 20 minutes while business loses money
```

**With Jenkins:**
```
Jenkins has all versions:
- v1.5.2 (current, broken)
- v1.5.1 (working)
- v1.5.0
- v1.4.9

Click "Deploy v1.5.1":
Jenkins: Automatically deploy previous version
Time: 2 minutes
Customers happy, weekend saved ✅
```

**Learning:**
- Jenkins keeps artifact history
- One-click rollback = save weekends

---

*Created by Karthik Reddy*

### Q: Design a sophisticated multi-branch pipeline with approval gates

**A:**
```groovy
// Jenkins Declarative Pipeline with approval gates

pipeline {
    agent any
    
    parameters {
        choice(name: 'ENVIRONMENT', choices: ['dev', 'staging', 'production'], description: 'Target environment')
        booleanParam(name: 'RUN_PERFORMANCE_TESTS', defaultValue: false, description: 'Run performance tests')
        string(name: 'VERSION', defaultValue: '1.0.0', description: 'Version to deploy')
    }
    
    options {
        timeout(time: 1, unit: 'HOURS')
        timestamps()
        buildDiscarder(logRotator(numToKeepStr: '50'))
        disableConcurrentBuilds()
    }
    
    environment {
        AWS_REGION = 'us-east-1'
        DOCKER_REGISTRY = 'myrepo.dkr.ecr.us-east-1.amazonaws.com'
        SLACK_WEBHOOK = credentials('slack-webhook')
    }
    
    stages {
        stage('Checkout') {
            steps {
                checkout scm
                script {
                    env.GIT_COMMIT_SHORT = sh(script: 'git rev-parse --short HEAD', returnStdout: true).trim()
                    env.GIT_BRANCH = sh(script: 'git rev-parse --abbrev-ref HEAD', returnStdout: true).trim()
                }
            }
        }
        
        stage('Build') {
            steps {
                script {
                    sh '''
                        echo "Building version ${VERSION}"
                        docker build -t ${DOCKER_REGISTRY}/myapp:${VERSION} .
                        docker tag ${DOCKER_REGISTRY}/myapp:${VERSION} ${DOCKER_REGISTRY}/myapp:latest
                    '''
                }
            }
        }
        
        stage('Test') {
            parallel {
                stage('Unit Tests') {
                    steps {
                        script {
                            sh 'npm test -- --coverage'
                            publishHTML([
                                reportDir: 'coverage',
                                reportFiles: 'index.html',
                                reportName: 'Coverage Report'
                            ])
                        }
                    }
                }
                
                stage('Integration Tests') {
                    steps {
                        sh 'npm run integration-test'
                    }
                }
                
                stage('Security Scan') {
                    steps {
                        sh '''
                            trivy image ${DOCKER_REGISTRY}/myapp:${VERSION}
                            npm audit
                        '''
                    }
                }
            }
        }
        
        stage('Performance Tests') {
            when {
                expression { params.RUN_PERFORMANCE_TESTS == true }
            }
            steps {
                sh 'npm run performance-test'
            }
        }
        
        stage('Push to Registry') {
            steps {
                script {
                    sh '''
                        aws ecr get-login-password --region ${AWS_REGION} | \
                        docker login --username AWS --password-stdin ${DOCKER_REGISTRY}
                        docker push ${DOCKER_REGISTRY}/myapp:${VERSION}
                        docker push ${DOCKER_REGISTRY}/myapp:latest
                    '''
                }
            }
        }
        
        stage('Deploy to Staging') {
            when {
                branch 'develop'
            }
            steps {
                script {
                    sh '''
                        helm upgrade --install myapp ./helm/myapp \
                            --values ./helm/values-staging.yaml \
                            --set image.tag=${VERSION} \
                            --namespace staging
                    '''
                }
                sleep(time: 2, unit: 'MINUTES')
                sh 'kubectl run smoke-tests --image=myapp-tests:latest -n staging'
            }
        }
        
        stage('Approval for Production') {
            when {
                branch 'main'
            }
            steps {
                script {
                    def userInput = input(
                        id: 'ProdDeploymentApproval',
                        message: 'Deploy to Production?',
                        parameters: [
                            string(name: 'APPROVER', description: 'Approver name'),
                            choice(name: 'DEPLOYMENT_TYPE', choices: ['BlueGreen', 'Canary', 'RollingUpdate'])
                        ]
                    )
                    
                    env.APPROVED_BY = userInput.APPROVER
                    env.DEPLOYMENT_TYPE = userInput.DEPLOYMENT_TYPE
                }
            }
        }
        
        stage('Deploy to Production') {
            when {
                branch 'main'
            }
            steps {
                script {
                    sh '''
                        helm upgrade --install myapp ./helm/myapp \
                            --values ./helm/values-prod.yaml \
                            --set image.tag=${VERSION} \
                            --set deploymentStrategy=${DEPLOYMENT_TYPE} \
                            --namespace production
                    '''
                }
                
                // Monitor deployment
                sh '''
                    kubectl rollout status deployment/myapp -n production --timeout=10m
                    
                    # Health checks
                    for i in {1..30}; do
                        if curl -f https://api.example.com/health; then
                            echo "Health check passed"
                            exit 0
                        fi
                        sleep 10
                    done
                    exit 1
                '''
            }
        }
    }
    
    post {
        always {
            script {
                // Generate reports
                junit 'test-results/**/*.xml'
                archiveArtifacts artifacts: 'dist/**', allowEmptyArchive: true
            }
        }
        
        success {
            script {
                def message = """
                ✅ Build Successful
                Version: ${VERSION}
                Environment: ${ENVIRONMENT}
                Approved By: ${env.APPROVED_BY ?: 'N/A'}
                """
                
                sh "curl -X POST -H 'Content-type: application/json' \
                    --data '{\"text\":\"${message}\"}' \
                    ${SLACK_WEBHOOK}"
            }
        }
        
        failure {
            script {
                def message = """
                ❌ Build Failed
                Job: ${JOB_NAME}
                Build: ${BUILD_NUMBER}
                Error: Check logs at ${BUILD_URL}
                """
                
                sh "curl -X POST -H 'Content-type: application/json' \
                    --data '{\"text\":\"${message}\"}' \
                    ${SLACK_WEBHOOK}"
            }
        }
    }
}
```

---

## 2. Performance & Scalability

### Q: Design a distributed Jenkins architecture for large-scale builds

**A:**
```groovy
// Jenkins distributed architecture

// Master Configuration (lightweight)
// - Only manages jobs
// - Delegates all execution to agents

// Agent Pool Architecture
/*
┌──────────────────────────┐
│   Jenkins Master         │
│   (Job coordination)     │
└──────────────────────────┘
           │
    ┌──────┼──────┬──────┐
    │      │      │      │
    ▼      ▼      ▼      ▼
  Build  Test  Deploy Docker
  Agent  Agents Agents Agents
  (5)    (10)   (5)    (20)
*/

// Kubernetes-based Jenkins
// Provision agents dynamically

def callPipeline(String jobName) {
    build job: jobName, parameters: [
        string(name: 'AGENT_LABEL', value: 'kubernetes'),
        string(name: 'DOCKER_IMAGE', value: 'myrepo/build-agent:latest')
    ]
}

// Pod template for dynamic agents
podTemplate(
    label: 'build-agent',
    containers: [
        containerTemplate(name: 'docker', image: 'docker:20.10', ttyEnabled: true),
        containerTemplate(name: 'kubectl', image: 'bitnami/kubectl:latest', ttyEnabled: true),
        containerTemplate(name: 'node', image: 'node:18', ttyEnabled: true)
    ]
) {
    node('build-agent') {
        container('docker') {
            sh 'docker build -t myapp:latest .'
        }
    }
}
```

---

## 3. Security & RBAC

### Q: Design enterprise-grade security for Jenkins

**A:**
```groovy
// Jenkins Security Configuration

// 1. Authentication (LDAP/OAuth)
import hudson.security.LDAPSecurityRealm
import jenkins.model.Jenkins

def instance = Jenkins.getInstance()

def ldapRealm = new LDAPSecurityRealm(
    'ldap.example.com',  // server
    null,                 // port (uses default 389)
    null,                 // rootDN
    'cn=admin,dc=example,dc=com',  // bindDN
    'password',           // bindPassword
    null,                 // userSearchBase
    '(uid={0})',         // userSearch
    null,                 // groupSearchBase
    null,                 // groupSearchFilter
    null,                 // manager DN
    false,               // ignore partial config
    false,               // disable mail address resolver
    null,                 // displayNameAttribute
    null,                 // mailAddressAttribute
    null                  // groupMembershipFilter
)

instance.setSecurityRealm(ldapRealm)
instance.save()

// 2. Authorization (Role-based)
import com.michelin.cio.hudson.plugins.rolestrategy.RoleBasedAuthorizationStrategy
import com.michelin.cio.hudson.plugins.rolestrategy.Role

def rbac = new RoleBasedAuthorizationStrategy()

// Define roles
def devRole = new Role('developer', [
    'hudson.model.Item.Build',
    'hudson.model.Item.View'
], true)

def adminRole = new Role('admin', [
    'hudson.model.Admin.ConfigureUpdateCenter',
    'hudson.model.Hudson.Administer'
], false)

rbac.addRole(RoleBasedAuthorizationStrategy.GLOBAL, adminRole)
rbac.addRole(RoleBasedAuthorizationStrategy.PROJECT, devRole)

instance.setAuthorizationStrategy(rbac)

// 3. Credentials Management
import com.cloudbees.plugins.credentials.CredentialsProvider
import com.cloudbees.plugins.credentials.domains.Domain
import org.jenkinsci.plugins.plaincredentials.impl.StringCredentialsImpl
import hudson.util.Secret

def store = Jenkins.instance.getExtensionList('com.cloudbees.plugins.credentials.SystemCredentialsProvider')[0].getStore()

def credential = new StringCredentialsImpl(
    CredentialsScope.GLOBAL,
    'docker-registry',
    'Docker Registry Credentials',
    Secret.fromString('mytoken')
)

store.addCredentials(Domain.global(), credential)

// 4. Audit Logging
import org.jenkinsci.plugins.audittrail.AuditTrailPlugin

def auditPlugin = Jenkins.instance.getPluginManager().getPlugin('audit-trail')
auditPlugin.setLogBuildCause(true)
auditPlugin.setLogConfigChanges(true)
```

---

## 4. Integration & Automation

### Q: Design comprehensive Jenkins integrations for DevOps

**A:**
```groovy
// Multi-tool Jenkins integration

pipeline {
    agent any
    
    stages {
        // Slack notifications
        stage('Slack Integration') {
            steps {
                script {
                    slackSend(
                        channel: '#ci-builds',
                        color: 'good',
                        message: "Build Started: ${JOB_NAME}",
                        tokenCredentialId: 'slack-token'
                    )
                }
            }
        }
        
        // GitHub integration
        stage('GitHub Check Runs') {
            steps {
                script {
                    checkRunsCreate(
                        name: 'Jenkins Build',
                        conclusion: 'neutral',
                        details_url: "${BUILD_URL}"
                    )
                }
            }
        }
        
        // Jira integration
        stage('Jira Update') {
            steps {
                script {
                    step([
                        $class: 'JiraIssueUpdater',
                        issueSelector: [$class: 'DefaultIssueSelector'],
                        scm: scm
                    ])
                    
                    jiraAddComment(
                        issue: 'PROJ-123',
                        comment: "Build #${BUILD_NUMBER} passed",
                        failOnError: false
                    )
                }
            }
        }
        
        // Artifactory integration
        stage('Publish to Artifactory') {
            steps {
                script {
                    rtUpload(
                        serverId: 'artifactory-server',
                        spec: '''{
                            "files": [{
                                "pattern": "dist/**",
                                "target": "libs-release/${VERSION}/"
                            }]
                        }'''
                    )
                }
            }
        }
        
        // Prometheus metrics
        stage('Publish Metrics') {
            steps {
                script {
                    sh '''
                        curl -X POST http://prometheus:9091/metrics/job/jenkins_job/instance/build_${BUILD_NUMBER} \
                            -d "jenkins_build_duration_seconds ${currentBuild.durationString}"
                    '''
                }
            }
        }
    }
}
```

---

## 5. Disaster Recovery

### Q: Design Jenkins HA with backup and recovery strategy

**A:**
```groovy
// Jenkins HA Architecture

/*
┌─────────────────────────────────────┐
│   Load Balancer (NGINX)             │
└─────────────────────────────────────┘
    ▲           ▲           ▲
    │           │           │
┌───┴──┐   ┌────┴──┐   ┌───┴──┐
│Jenkins│   │Jenkins│   │Jenkins│
│Master1│   │Master2│   │Master3│
└───┬──┘   └────┬──┘   └───┬──┘
    │           │           │
    └───────┬───┴───┬───────┘
            │       │
     ┌──────▼─┐  ┌──▼──────┐
     │ Consul │  │Etcd Bkup│
     │ Shared │  │Database │
     │ Config │  │(S3 Sync)│
     └────────┘  └─────────┘
*/

// Backup Strategy
@NonCPS
void backupJenkins() {
    sh '''
        # Backup Jenkins home
        tar -czf /backups/jenkins-home-$(date +%Y%m%d-%H%M%S).tar.gz \
            /var/lib/jenkins \
            --exclude=workspace \
            --exclude=plugins/\* \
            --exclude=.cache
        
        # Upload to S3
        aws s3 cp /backups/*.tar.gz s3://jenkins-backups/
        
        # Keep only last 30 days
        find /backups -name "*.tar.gz" -mtime +30 -delete
    '''
}

// Recovery
@NonCPS
void restoreJenkins(String backupFile) {
    sh '''
        systemctl stop jenkins
        
        # Restore from backup
        tar -xzf ${backupFile} -C /
        
        # Restore plugins
        cd /var/lib/jenkins/plugins
        java -jar jenkins-cli.jar -s http://localhost:8080 install-plugin \
            $(cat /var/lib/jenkins/plugins.txt)
        
        systemctl start jenkins
    '''
}
```

---

## Summary

**Critical Senior Jenkins Skills:**
1. ✅ Pipeline design & declarative syntax
2. ✅ Distributed architectures
3. ✅ Security & authentication
4. ✅ Multi-tool integrations
5. ✅ Performance optimization
6. ✅ HA & disaster recovery
7. ✅ Custom plugins & extensions
8. ✅ GitOps integration

*Created by Karthik Reddy*

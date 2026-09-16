# Jenkins Cheat Sheet

> Jenkins pipeline reference — declarative and scripted pipelines, agents, stages, credentials, shared libraries, and common steps.

---

## Table of Contents

- [Declarative Pipeline Skeleton](#1-declarative-pipeline-skeleton)
- [Directives: agent, options, environment, parameters](#2-core-directives)
- [Stages & Steps](#3-stages--steps)
- [Post Actions](#4-post-actions)
- [Credentials & Secrets](#5-credentials--secrets)
- [Common Steps](#6-common-steps)
- [Scripted Pipeline](#7-scripted-pipeline)
- [Triggers](#8-triggers)
- [Shared Libraries](#9-shared-libraries)
- [Environment Variables](#10-environment-variables)
- [Jenkins CLI & Useful Groovy](#11-jenkins-cli--useful-groovy)

---

## 1. Declarative Pipeline Skeleton

```groovy
pipeline {
    agent any

    options {
        timestamps()
        disableConcurrentBuilds()
        buildDiscarder(logRotator(numToKeepStr: '20'))
        timeout(time: 30, unit: 'MINUTES')
    }

    environment {
        APP_NAME = 'myapp'
        DOCKER_IMAGE = "registry.example.com/${APP_NAME}:${BUILD_NUMBER}"
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }
        stage('Build') {
            steps {
                sh 'make build'
            }
        }
        stage('Test') {
            steps {
                sh 'make test'
            }
            post {
                always {
                    junit '**/target/surefire-reports/*.xml'
                }
            }
        }
        stage('Deploy') {
            when {
                branch 'main'
            }
            steps {
                sh 'make deploy'
            }
        }
    }

    post {
        success { echo 'Pipeline succeeded!' }
        failure { echo 'Pipeline failed!' }
    }
}
```

## 2. Core Directives

```groovy
// Agent — where the pipeline runs
agent any
agent none                                    // stage-level agents required
agent { label 'linux && docker' }             // node label expression
agent { node { label 'linux'; customWorkspace '/opt/build' } }
agent {
    docker { image 'node:20-alpine'; args '-u root'; label 'linux' }
}
agent {
    kubernetes {
        yaml '''
spec:
  containers:
  - name: build
    image: maven:3.9-eclipse-temurin-21
'''
    }
}

// Options
options {
    timestamps()
    ansiColor('xterm')
    disableConcurrentBuilds()
    skipStagesAfterUnstable()
    retry(2)
    buildDiscarder(logRotator(daysToKeepStr: '30', numToKeepStr: '50'))
    timeout(time: 1, unit: 'HOURS')
}

// Parameters (Build with Parameters UI)
parameters {
    string(name: 'VERSION', defaultValue: '1.0.0', description: 'Release version')
    choice(name: 'ENV', choices: ['dev', 'staging', 'prod'], description: 'Target env')
    booleanParam(name: 'RUN_TESTS', defaultValue: true)
    password(name: 'API_KEY', description: 'External API key')
}

// Environment variables + credentials
environment {
    CI = 'true'
    REGISTRY = 'registry.example.com'
    REGISTRY_CREDS = credentials('registry-creds-id')     // Creates REGISTRY_CREDS_USR / _PSW
}
```

## 3. Stages & Steps

```groovy
stages {
    stage('Parallel Tests') {
        parallel {
            stage('Unit Tests') {
                steps { sh 'npm test' }
            }
            stage('Lint') {
                steps { sh 'npm run lint' }
            }
            stage('Security Scan') {
                steps { sh 'trivy fs .' }
            }
        }
    }

    stage('Matrix Build') {
        matrix {
            axes {
                axis {
                    name 'PLATFORM'
                    values 'linux/amd64', 'linux/arm64'
                }
                axis {
                    name 'NODE_VERSION'
                    values '18', '20'
                }
            }
            stages {
                stage('Build') {
                    steps {
                        echo "Building ${PLATFORM} on Node ${NODE_VERSION}"
                    }
                }
            }
        }
    }

    stage('Conditional Deploy') {
        when {
            allOf {
                branch 'main'
                expression { params.ENV == 'prod' }
                environment name: 'DEPLOY_ALLOWED', value: 'true'
            }
        }
        steps { sh './deploy.sh' }
    }
}
```

**`when` conditions:** `branch 'main'`, `buildingTag()`, `tag 'v*'`, `changeset '**/*.js'`, `expression { ... }`, `environment name: 'X', value: 'Y'`, `not / anyOf / allOf`.

## 4. Post Actions

```groovy
post {
    always  { cleanWs() }                                  // Clean workspace
    success { echo 'OK' }
    failure {
        mail to: 'team@example.com', subject: "FAILED: ${JOB_NAME} #${BUILD_NUMBER}",
             body: "See ${BUILD_URL}"
    }
    unstable { echo 'Tests flaky' }
    aborted  { echo 'Aborted' }
    changed  { echo 'Status changed from previous build' }
}
```

## 5. Credentials & Secrets

```groovy
// Using credentials in a single stage
stage('Push Image') {
    steps {
        withCredentials([usernamePassword(
            credentialsId: 'docker-registry',
            usernameVariable: 'REG_USER',
            passwordVariable: 'REG_PASS')]) {
            sh '''
                echo "$REG_PASS" | docker login registry.example.com -u "$REG_USER" --password-stdin
                docker push $DOCKER_IMAGE
            '''
        }

        withCredentials([sshUserPrivateKey(
            credentialsId: 'deploy-key',
            keyFileVariable: 'SSH_KEY')]) {
            sh 'ssh -i $SSH_KEY -o StrictHostKeyChecking=no deploy@prod ./release.sh'
        }

        withCredentials([string(credentialsId: 'api-token', variable: 'TOKEN')]) {
            sh 'curl -H "Authorization: Bearer $TOKEN" https://api.example.com/deploy'
        }
    }
}
```

**Never** echo secrets; use `set +x` or shell single quotes to avoid credential masking leaks.

## 6. Common Steps

```groovy
steps {
    checkout scm                                    // Clone this job's repo
    git branch: 'main', url: 'https://github.com/org/repo.git'
    sh 'mvn clean package'                          // Shell (Unix)
    sh '''
        set -e
        make build
        make test
    '''
    bat 'build.bat'                                 // Windows batch
    powershell 'Get-ChildItem'
    echo "Building version ${params.VERSION}"
    script {                                        // Run Groovy inside declarative
        def msg = "Deploying to ${params.ENV}"
        echo msg
        env.DEPLOY_MSG = msg
    }
    dir('subproject') { sh 'npm install' }
    withEnv(['MY_VAR=abc']) { sh 'echo $MY_VAR' }
    retry(3) { sh 'curl -f https://api.example.com/health' }
    timeout(time: 5, unit: 'MINUTES') { sh 'long-task.sh' }
    sleep(time: 30, unit: 'SECONDS')
    input message: 'Deploy to production?', ok: 'Deploy',
          parameters: [choice(name: 'TARGET', choices: 'blue\ngreen')]
    stash(name: 'artifacts', includes: '**/target/*.jar')
    unstash 'artifacts'
    archiveArtifacts artifacts: '**/target/*.jar', fingerprint: true
    junit(testResults: '**/target/surefire-reports/*.xml', allowEmptyResults: true)
    publishHTML(target: [reportDir: 'reports', reportFiles: 'index.html', reportName: 'Coverage'])
    cleanWs()
}
```

## 7. Scripted Pipeline

```groovy
node('linux') {
    try {
        stage('Checkout') {
            checkout scm
        }
        stage('Build') {
            docker.image('maven:3.9').inside('-v $HOME/.m2:/root/.m2') {
                sh 'mvn clean package'
            }
        }
        def imageTag = "app:${env.BUILD_NUMBER}"
        stage('Push') {
            withCredentials([usernamePassword(credentialsId: 'reg',
                    usernameVariable: 'U', passwordVariable: 'P')]) {
                sh "docker login -u $U -p $P registry.example.com"
                sh "docker push registry.example.com/${imageTag}"
            }
        }
        currentBuild.result = 'SUCCESS'
    } catch (err) {
        currentBuild.result = 'FAILURE'
        throw err
    } finally {
        cleanWs()
    }
}
```

## 8. Triggers

```groovy
triggers {
    cron('H 2 * * *')                       # Scheduled (daily 2 AM, hashed minute)
    pollSCM('H/5 * * * *')                  # Poll SCM every 5 minutes
    upstream(upstreamProjects: 'lib-build', threshold: hudson.model.Result.SUCCESS)
}
```

Also: **GitHub/GitLab/Bitbucket webhooks** (via multibranch pipelines), **Generic Webhook Trigger plugin**.

**Multibranch pipeline:** creates a job per branch, auto-discovers `Jenkinsfile`, PRs build with `CHANGE_ID`.

## 9. Shared Libraries

`vars/deployApp.groovy`:
```groovy
def call(String env, String version) {
    echo "Deploying ${version} to ${env}"
    sh "./deploy.sh ${env} ${version}"
}
```

`src/org/foo/Utils.groovy` — Groovy classes for complex logic.

Usage:
```groovy
@Library('my-shared-lib') _            // Load implicit library
def utils = new org.foo.Utils()
pipeline {
    agent any
    stages {
        stage('Deploy') {
            steps { deployApp('prod', params.VERSION) }
        }
    }
}
```

## 10. Environment Variables

| Variable | Description |
| --- | --- |
| `BUILD_NUMBER` | Build number |
| `BUILD_ID` | Build ID (timestamp-based in pipelines) |
| `BUILD_URL` | Absolute URL of the build |
| `JOB_NAME` / `JOB_URL` | Job name / URL |
| `WORKSPACE` | Absolute workspace path |
| `GIT_BRANCH` / `BRANCH_NAME` | Branch (multibranch) |
| `GIT_COMMIT` | Commit SHA |
| `CHANGE_ID` | PR number (multibranch) |
| `EXECUTOR_NUMBER` | Executor slot on agent |

## 11. Jenkins CLI & Useful Groovy

```bash
# Jenkins CLI (requires API token)
java -jar jenkins-cli.jar -s http://jenkins:8080 -auth user:API_TOKEN who-am-i
java -jar jenkins-cli.jar -s http://jenkins:8080 build my-job -p VERSION=1.2.3

# Restart / safe-restart
curl -X POST -u user:token http://jenkins:8080/safeRestart
```

```groovy
// Script Console snippets (Manage Jenkins → Script Console)
Jenkins.instance.getJobNames().each { println it }                    // List jobs
Jenkins.instance.itemMap.each { k, v -> println "$k: ${v.lastBuild?.result}" }
```

---

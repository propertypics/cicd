pipeline {
    agent any
    
    environment {
        NODE_VERSION = '18'
        DOCKER_REGISTRY = 'your-registry.com'
        APP_NAME = 'express-cicd-sandbox'
        KUBECONFIG = credentials('kubeconfig')
    }
    
    options {
        buildDiscarder(logRotator(numToKeepStr: '10'))
        timeout(time: 30, unit: 'MINUTES')
        timestamps()
    }
    
    tools {
        nodejs "${NODE_VERSION}"
    }
    
    stages {
        stage('Checkout') {
            steps {
                checkout scm
                script {
                    env.GIT_COMMIT_SHORT = sh(
                        script: 'git rev-parse --short HEAD',
                        returnStdout: true
                    ).trim()
                    env.BUILD_VERSION = "${env.BUILD_NUMBER}-${env.GIT_COMMIT_SHORT}"
                }
            }
        }
        
        stage('Install Dependencies') {
            steps {
                sh 'npm ci'
            }
        }
        
        stage('Code Quality') {
            parallel {
                stage('Lint') {
                    steps {
                        sh 'npm run lint'
                    }
                }
                stage('Security Audit') {
                    steps {
                        sh 'npm audit --audit-level=moderate'
                    }
                }
            }
        }
        
        stage('Test') {
            steps {
                sh 'npm run test:coverage'
            }
            post {
                always {
                    publishHTML([
                        allowMissing: false,
                        alwaysLinkToLastBuild: false,
                        keepAll: true,
                        reportDir: 'coverage',
                        reportFiles: 'index.html',
                        reportName: 'Coverage Report'
                    ])
                }
            }
        }
        
        stage('Build Docker Image') {
            when {
                anyOf {
                    branch 'main'
                    branch 'develop'
                }
            }
            steps {
                script {
                    def image = docker.build("${DOCKER_REGISTRY}/${APP_NAME}:${BUILD_VERSION}")
                    docker.withRegistry("https://${DOCKER_REGISTRY}", 'docker-registry-credentials') {
                        image.push()
                        image.push('latest')
                    }
                }
            }
        }
        
        stage('Security Scan') {
            when {
                anyOf {
                    branch 'main'
                    branch 'develop'
                }
            }
            steps {
                script {
                    sh """
                        docker run --rm -v /var/run/docker.sock:/var/run/docker.sock \\
                        -v \$PWD/cache:/root/.cache/ \\
                        aquasec/trivy:latest image \\
                        --exit-code 0 \\
                        --severity HIGH,CRITICAL \\
                        --format json \\
                        -o trivy-report.json \\
                        ${DOCKER_REGISTRY}/${APP_NAME}:${BUILD_VERSION}
                    """
                }
            }
            post {
                always {
                    archiveArtifacts artifacts: 'trivy-report.json', fingerprint: true
                }
            }
        }
        
        stage('Deploy to Staging') {
            when {
                branch 'develop'
            }
            steps {
                script {
                    sh """
                        helm upgrade --install ${APP_NAME}-staging ./helm-chart \\
                        --namespace staging \\
                        --create-namespace \\
                        --set image.repository=${DOCKER_REGISTRY}/${APP_NAME} \\
                        --set image.tag=${BUILD_VERSION} \\
                        --set environment=staging \\
                        --wait --timeout=300s
                    """
                }
            }
            post {
                success {
                    script {
                        // Run smoke tests
                        sh """
                            sleep 30  # Wait for deployment to stabilize
                            curl -f https://staging.${APP_NAME}.com/health || exit 1
                        """
                    }
                }
            }
        }
        
        stage('Deploy to Production') {
            when {
                branch 'main'
            }
            steps {
                script {
                    // Manual approval for production deployment
                    timeout(time: 5, unit: 'MINUTES') {
                        input message: 'Deploy to production?', ok: 'Deploy',
                              submitterParameter: 'DEPLOYER'
                    }
                    
                    sh """
                        helm upgrade --install ${APP_NAME}-prod ./helm-chart \\
                        --namespace production \\
                        --create-namespace \\
                        --set image.repository=${DOCKER_REGISTRY}/${APP_NAME} \\
                        --set image.tag=${BUILD_VERSION} \\
                        --set environment=production \\
                        --set replicaCount=3 \\
                        --wait --timeout=300s
                    """
                }
            }
            post {
                success {
                    script {
                        // Run smoke tests
                        sh """
                            sleep 30  # Wait for deployment to stabilize
                            curl -f https://${APP_NAME}.com/health || exit 1
                        """
                    }
                }
            }
        }
    }
    
    post {
        always {
            cleanWs()
        }
        success {
            script {
                if (env.BRANCH_NAME == 'main') {
                    slackSend(
                        channel: '#deployments',
                        color: 'good',
                        message: "✅ ${APP_NAME} v${BUILD_VERSION} deployed to production by ${env.DEPLOYER}"
                    )
                }
            }
        }
        failure {
            script {
                slackSend(
                    channel: '#deployments',
                    color: 'danger',
                    message: "❌ ${APP_NAME} build ${BUILD_NUMBER} failed on ${env.BRANCH_NAME}"
                )
            }
        }
    }
}

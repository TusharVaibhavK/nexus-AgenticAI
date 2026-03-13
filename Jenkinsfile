pipeline {
    agent any

    tools {
        dockerTool 'docker'
    }

    triggers {
        githubPush()
    }

    environment {
        DOCKER_CREDENTIALS_ID = 'docker-cred'
        DOCKER_USERNAME      = 'trahulprabhu38'
        DOCKER_REGISTRY      = 'https://index.docker.io/v1/'
        K8S_NAMESPACE        = 'default'
        GIT_COMMIT_SHORT     = sh(script: "git rev-parse --short HEAD", returnStdout: true).trim()
        BUILD_TAG            = "${env.BUILD_NUMBER}-${GIT_COMMIT_SHORT}"
        IMAGE_TAG            = 'latest'
    }

    parameters {
        booleanParam(name: 'BUILD_SQL_QUERY_GEN',  defaultValue: true,  description: 'Build SQL Query Generator')
        booleanParam(name: 'BUILD_INTENT_AGENT',   defaultValue: true,  description: 'Build Intent Agent')
        booleanParam(name: 'BUILD_SQL_VALIDATOR',  defaultValue: true,  description: 'Build SQL Validator Agent')
        booleanParam(name: 'BUILD_COLUMN_PRUNING', defaultValue: true,  description: 'Build Column Pruning')
        booleanParam(name: 'BUILD_FRONTEND',       defaultValue: true,  description: 'Build Frontend')
        booleanParam(name: 'PUSH_TO_REGISTRY',     defaultValue: true,  description: 'Push images to Docker registry')
        booleanParam(name: 'DEPLOY_TO_K8S',        defaultValue: false, description: 'Deploy to Kubernetes cluster')
        choice(name: 'TAG_VERSION', choices: ['latest', 'v1', 'v2', 'custom'], description: 'Docker image tag version')
        string(name: 'CUSTOM_TAG', defaultValue: '', description: 'Custom tag (if TAG_VERSION is custom)')
    }

    stages {
        stage('Git Checkout') {
            steps {
                git branch: 'main',
                    credentialsId: 'git-cred',
                    url: 'https://github.com/trahulprabhu38/nexus-AgenticAI'
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonarqube') {
                    script {
                        def scannerHome = tool 'sonarqube'
                        sh """
                            ${scannerHome}/bin/sonar-scanner \
                              -Dsonar.projectKey=nexus-agentic-ai \
                              -Dsonar.projectName=nexus-agentic-ai \
                              -Dsonar.sources=. \
                              -Dsonar.language=py \
                              -Dsonar.python.version=3
                        """
                    }
                }
            }
        }

        stage('Quality Gate') {
            steps {
                timeout(time: 3, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Environment Setup') {
            steps {
                script {
                    env.IMAGE_TAG = (params.TAG_VERSION == 'custom' && params.CUSTOM_TAG?.trim()) ? params.CUSTOM_TAG : params.TAG_VERSION
                    echo "Docker Image Tag: ${env.IMAGE_TAG}"
                    echo "Build Tag: ${env.BUILD_TAG}"
                }
            }
        }

        stage('Build Docker Images') {
            parallel {
                stage('SQL Query Generator') {
                    when { expression { params.BUILD_SQL_QUERY_GEN } }
                    steps {
                        script {
                            dir('SQL_QUERY_GENERATOR') {
                                docker.withRegistry(env.DOCKER_REGISTRY, env.DOCKER_CREDENTIALS_ID) {
                                    def img = docker.build(
                                        "${env.DOCKER_USERNAME}/sql-query-gen:${env.IMAGE_TAG}",
                                        "--build-arg BUILD_DATE=\$(date -u +'%Y-%m-%dT%H:%M:%SZ') --build-arg VCS_REF=${env.GIT_COMMIT_SHORT} ."
                                    )
                                    if (params.PUSH_TO_REGISTRY) {
                                        img.push()
                                        img.push(env.BUILD_TAG)
                                    }
                                }
                            }
                        }
                    }
                }

                stage('Intent Agent') {
                    when { expression { params.BUILD_INTENT_AGENT } }
                    steps {
                        script {
                            dir('Intent-Agent') {
                                docker.withRegistry(env.DOCKER_REGISTRY, env.DOCKER_CREDENTIALS_ID) {
                                    def img = docker.build(
                                        "${env.DOCKER_USERNAME}/intent-agent:${env.IMAGE_TAG}",
                                        "--build-arg BUILD_DATE=\$(date -u +'%Y-%m-%dT%H:%M:%SZ') --build-arg VCS_REF=${env.GIT_COMMIT_SHORT} ."
                                    )
                                    if (params.PUSH_TO_REGISTRY) {
                                        img.push()
                                        img.push(env.BUILD_TAG)
                                    }
                                }
                            }
                        }
                    }
                }

                stage('SQL Validator Agent') {
                    when { expression { params.BUILD_SQL_VALIDATOR } }
                    steps {
                        script {
                            dir('sql_validator_agent') {
                                docker.withRegistry(env.DOCKER_REGISTRY, env.DOCKER_CREDENTIALS_ID) {
                                    def img = docker.build(
                                        "${env.DOCKER_USERNAME}/sql-validator:${env.IMAGE_TAG}",
                                        "--build-arg BUILD_DATE=\$(date -u +'%Y-%m-%dT%H:%M:%SZ') --build-arg VCS_REF=${env.GIT_COMMIT_SHORT} ."
                                    )
                                    if (params.PUSH_TO_REGISTRY) {
                                        img.push()
                                        img.push(env.BUILD_TAG)
                                    }
                                }
                            }
                        }
                    }
                }

                stage('Column Pruning') {
                    when { expression { params.BUILD_COLUMN_PRUNING } }
                    steps {
                        script {
                            dir('column pruning') {
                                docker.withRegistry(env.DOCKER_REGISTRY, env.DOCKER_CREDENTIALS_ID) {
                                    def img = docker.build(
                                        "${env.DOCKER_USERNAME}/column-prune:${env.IMAGE_TAG}",
                                        "--build-arg BUILD_DATE=\$(date -u +'%Y-%m-%dT%H:%M:%SZ') --build-arg VCS_REF=${env.GIT_COMMIT_SHORT} ."
                                    )
                                    if (params.PUSH_TO_REGISTRY) {
                                        img.push()
                                        img.push(env.BUILD_TAG)
                                    }
                                }
                            }
                        }
                    }
                }

                stage('Frontend') {
                    when { expression { params.BUILD_FRONTEND } }
                    steps {
                        script {
                            dir('frontend') {
                                docker.withRegistry(env.DOCKER_REGISTRY, env.DOCKER_CREDENTIALS_ID) {
                                    def img = docker.build(
                                        "${env.DOCKER_USERNAME}/nexus-frontend:${env.IMAGE_TAG}",
                                        "--build-arg BUILD_DATE=\$(date -u +'%Y-%m-%dT%H:%M:%SZ') --build-arg VCS_REF=${env.GIT_COMMIT_SHORT} ."
                                    )
                                    if (params.PUSH_TO_REGISTRY) {
                                        img.push()
                                        img.push(env.BUILD_TAG)
                                    }
                                }
                            }
                        }
                    }
                }
            }
        }

        stage('Deploy to Kubernetes') {
            when { expression { params.DEPLOY_TO_K8S } }
            steps {
                script {
                    echo "Deploying to Kubernetes..."
                    sh """
                        if [ "${params.BUILD_SQL_QUERY_GEN}" = "true" ]; then
                            kubectl set image deployment/sql-query-gen-deployment \
                                sql-query-gen=${env.DOCKER_USERNAME}/sql-query-gen:${env.IMAGE_TAG} \
                                -n ${env.K8S_NAMESPACE}
                        fi
                        if [ "${params.BUILD_INTENT_AGENT}" = "true" ]; then
                            kubectl set image deployment/intent-agent-deployment \
                                intent-agent=${env.DOCKER_USERNAME}/intent-agent:${env.IMAGE_TAG} \
                                -n ${env.K8S_NAMESPACE}
                        fi
                        if [ "${params.BUILD_SQL_VALIDATOR}" = "true" ]; then
                            kubectl set image deployment/sql-validator-api-deployment \
                                sql-validator=${env.DOCKER_USERNAME}/sql-validator:${env.IMAGE_TAG} \
                                -n ${env.K8S_NAMESPACE}
                        fi
                        if [ "${params.BUILD_COLUMN_PRUNING}" = "true" ]; then
                            kubectl set image deployment/column-prune-deployment \
                                column-prune=${env.DOCKER_USERNAME}/column-prune:${env.IMAGE_TAG} \
                                -n ${env.K8S_NAMESPACE}
                        fi
                        if [ "${params.BUILD_FRONTEND}" = "true" ]; then
                            kubectl set image deployment/frontend-deployment \
                                frontend=${env.DOCKER_USERNAME}/nexus-frontend:${env.IMAGE_TAG} \
                                -n ${env.K8S_NAMESPACE}
                        fi
                    """
                    echo "Waiting for rollouts to complete..."
                    sh """
                        kubectl rollout status deployment/sql-query-gen-deployment    -n ${env.K8S_NAMESPACE} --timeout=5m || true
                        kubectl rollout status deployment/intent-agent-deployment     -n ${env.K8S_NAMESPACE} --timeout=5m || true
                        kubectl rollout status deployment/sql-validator-api-deployment -n ${env.K8S_NAMESPACE} --timeout=5m || true
                        kubectl rollout status deployment/column-prune-deployment     -n ${env.K8S_NAMESPACE} --timeout=5m || true
                        kubectl rollout status deployment/frontend-deployment         -n ${env.K8S_NAMESPACE} --timeout=5m || true
                    """
                }
            }
        }

        stage('Verify Deployment') {
            when { expression { params.DEPLOY_TO_K8S } }
            steps {
                sh """
                    echo "=== Deployment Status ==="
                    kubectl get deployments -n ${env.K8S_NAMESPACE}
                    echo "=== Pod Status ==="
                    kubectl get pods -n ${env.K8S_NAMESPACE}
                    echo "=== Service Status ==="
                    kubectl get services -n ${env.K8S_NAMESPACE}
                """
            }
        }
    }

    post {
        always {
            echo "Pipeline execution completed"
        }
        success {
            script {
                sh 'docker image prune -f || true'
                echo "Build and push successful!"
            }
        }
        failure {
            echo "Pipeline failed. Check stage logs above for details."
        }
        cleanup {
            cleanWs()
        }
    }
}
